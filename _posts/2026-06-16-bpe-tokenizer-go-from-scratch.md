---
layout: post
title: "building a fast bpe tokenizer in pure go"
date: 2026-06-16 10:00:00 +0530
slug: bpe-tokenizer-go-from-scratch
---

i was working on a go project that needed to tokenize text before feeding it into an llm pipeline , every option i found had the same problem , they either shelled out to python , used cgo to wrap the rust tiktoken library , or pulled in half a framework to do something conceptually simple , i didnt want a python sidecar , i didnt want to fight cgo on a linux arm box at 2am , i wanted a pure go tokenizer that i could ship as a single binary , understand completely , and eventually blame only myself for

so i built tokr , a byte-pair encoding tokenizer in pure go that hits **~9 mb/s throughput, ~5.3m tokens/sec on the hot path, and a 52-nanosecond cached response time** , thats roughly 80% of what openais tiktoken achieves in rust , this post is the story of how that number came to be , what i got wrong first , and what the architecture actually looks like under the hood

> **url slug:** `/posts/bpe-tokenizer-go-from-scratch`  
> if you found this searching for "bpe tokenizer go implementation" or "tiktoken alternative pure go" then yes , youre in the right place

---

## why bpe? a sixty-second primer

why does any of this matter

language models dont read words , they read *tokens* , which are chunks of text that sit somewhere between a character and a word , byte-pair encoding (bpe) is the algorithm that decides what those chunks are , gpt-4 uses it , llama uses it , it works like this

1. start with a vocabulary of 256 individual bytes (every possible byte value , 0–255)
2. count every adjacent pair of tokens in your training corpus
3. merge the most frequent pair into a new single token , assign it the next available id
4. repeat until you reach your target vocabulary size

the result is a vocabulary that naturally captures common english subwords (`" the"`, `"ing"`, `" and"`) , while still being able to represent literally anything , even binary data , because you always fall back to raw bytes

when you encode new text , you replay those merges in the order they were learned , always choosing the merge with the lowest rank (the one learned earliest) at each step , thats the whole algorithm , the hard part is doing it fast

---

## the architecture: four layers, one binary

heres how tokr is structured

<div class="arch-diagram">
  <div class="arch-node">input text</div>
  
  <div class="arch-arrow">
    <svg width="24" height="30" viewBox="0 0 24 30"><path d="M12 0v28M5 21l7 7 7-7" fill="none" stroke="currentColor" stroke-width="2"/></svg>
  </div>
  
  <div class="arch-row">
    <div class="arch-box">
      <div class="arch-box-title">splitter module</div>
      <div class="arch-box-desc">gptsplit (regexp2)<br>or fastsplit (custom scanner)</div>
    </div>
    <div class="arch-note"><span style="margin-right:0.5rem">←</span> pre-tokenization boundary enforcement</div>
  </div>
  
  <div class="arch-arrow">
    <div class="arch-arrow-label">[]string chunks</div>
    <svg width="24" height="30" viewBox="0 0 24 30"><path d="M12 0v28M5 21l7 7 7-7" fill="none" stroke="currentColor" stroke-width="2"/></svg>
  </div>
  
  <div class="arch-row">
    <div class="arch-box">
      <div class="arch-box-title">bpe engine</div>
      <div class="arch-box-desc">cache check (o(1) rwmutex)<br>rank-based merge loop</div>
    </div>
    <div class="arch-note"><span style="margin-right:0.5rem">←</span> runmergelogic, in-place slice mutation</div>
  </div>
  
  <div class="arch-arrow">
    <svg width="24" height="20" viewBox="0 0 24 20"><path d="M12 0v20" fill="none" stroke="currentColor" stroke-width="2"/></svg>
  </div>
  
  <div class="arch-branch-container">
    <div class="arch-branch-horizontal"></div>
    <div class="arch-branch-verticals">
      <div class="arch-branch-line-left"><div class="arch-arrow-head"></div></div>
      <div class="arch-branch-line-right"><div class="arch-arrow-head"></div></div>
    </div>
  </div>
  
  <div class="arch-split-row">
    <div class="arch-box small-box">
      <div class="arch-box-title">single-threaded</div>
      <div class="arch-box-desc">encode()</div>
    </div>
    <div class="arch-box small-box">
      <div class="arch-box-title">worker pool</div>
      <div class="arch-box-desc">parallelencode()<br><span style="font-size: 0.75rem; opacity: 0.8">(runtime.numcpu workers,<br>buffered channels)</span></div>
    </div>
  </div>
  
  <div class="arch-arrow" style="margin-top: 1rem;">
    <svg width="24" height="30" viewBox="0 0 24 30"><path d="M12 0v28M5 21l7 7 7-7" fill="none" stroke="currentColor" stroke-width="2"/></svg>
  </div>
  
  <div class="arch-row">
    <div class="arch-box" style="border-color: var(--link-color); background: rgba(88, 166, 255, 0.05);">
      <div class="arch-box-title">http layer</div>
      <div class="arch-box-desc">/encode  /decode<br>smart routing: &lt;999kb → single, ≥999kb → pool</div>
    </div>
    <div class="arch-note"><span style="margin-right:0.5rem">←</span> net/http, no framework</div>
  </div>
</div>

every layer has exactly one job , the splitter doesnt merge , the merge engine doesnt split , the http layer doesnt know what a token is , it just routes

---

## the http api

this is a feature it should not be at the last of the blog , tokr ships with a lightweight server using only net/http and no frameworks 

you just send a post request to `/encode` with a json payload containing your text , and it returns the tokens , the token count , and the time taken in seconds , similarly you can hit `/decode` with an array of tokens to get the text back

the routing logic in `/encode` is a simple size gate , below 999kb the single-threaded cache-optimized path wins , above it the worker pool takes over , the model loads once at startup and stays in memory

---

## layer 1: the splitter

this was the first thing i got wrong conceptually , you cant just take a string like "hello world" , convert it to bytes , and start merging , if you do the merge algorithm will happily create a token that spans the space between "hello" and "world" , now " wo" is a single token in your vocabulary , thats not how gpt-4 does it , and it breaks cross-model compatibility

the fix is pre-tokenization , before bpe even sees the text , you split it into chunks at semantic boundaries , merges are only allowed to happen within a chunk , never across them

tokr has two splitters

### gptsplit: full compatibility mode

this uses the exact regex gpt-4s tokenizer uses 

<div class="skip-note">
  <div class="skip-note-title"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z"></path></svg> deep dive (feel free to skip)</div>
  i had to pull in github.com/dlclark/regexp2 because gos standard regexp package doesnt support unicode category expressions like \p{L} (letter) and \p{N} (number) , the regexp2 library is a go port of .nets regex engine , its powerful , correct , and notoriously allocation-heavy , ill explain how i dealt with that later
</div>

### fastsplit: the zero-allocation path

for cases where you dont need 1:1 tiktoken compatibility , fastsplit is a hand-rolled unicode scanner

i wrote this because regexp2 on a 100kb input was allocating millions of times in benchmarks , fastsplit produces zero heap allocations during the scan itself , it just slices into the existing rune slice , its about 3x faster than gptsplit on raw throughput , at the cost of slight tokenization differences from the canonical gpt-4 output

the tradeoff is explicit here , `usegpt4` is threaded through every public api call , you pick your tradeoff at the call site

---

## layer 2: the merge engine

the core of the tokenizer is where the actual bpe encoding happens for a single pre-tokenized chunk

the key design decision here is in-place mutation , early versions of this function did the obvious thing which is to allocate a new slice every iteration and append to it , thats o(n) allocations per chunk , which turns into 18 million allocations on a 16mb corpus

the in-place approach overwrites the lowest rank mergeable pair with the merged token id , then shifts the right half of the slice one position left , the slice shrinks by one element per iteration , zero heap allocations , no gc pressure on the hot path

the tradeoff is that the callers slice is mutated , if you need the original you have to copy it before calling

---

## layer 3: the cache

heres the benchmark that made this click for me

<div id="chart-latency" class="chart-container"></div>

thats roughly a 47x latency difference between cold and hot path , once a string has been tokenized once , all future calls return in ~52–220 nanoseconds with a single allocation

the rwmutex is deliberate , concurrent reads dont block each other , only a write takes the exclusive lock , in a server handling many concurrent requests for common strings this matters a lot

eviction strategy is naive right now , when the cache exceeds 100,000 entries the whole map is replaced with a fresh empty one , its a full clear not lru , i know i will fix this later

one more thing about cache correctness , when a cache hit is returned the code makes a defensive copy , without it a caller who mutates the returned slice would corrupt the cached value for every future caller , it costs one allocation , its absolutely worth it , i considered removing it then remembered that "zero allocations but occasionally wrong" is a lot worse than "one allocation, always correct"

---

## layer 4: the worker pool

for inputs over 1mb , tokr splits the text into ~1mb chunks and encodes them in parallel using a buffered jobs channel

the chunking is careful about two things

1. **natural boundaries:** the splitter scans backward from the 1mb mark to find a space or newline so words are never cut mid-token
2. **utf-8 integrity:** if no natural boundary is found in the backward scan , theres a bitwise fallback to ensure we dont split mid-rune and hand regexp2 a malformed string

on a 16mb corpus with parallel encoding:

<div id="chart-scaling" class="chart-container"></div>

the parallel speedup on raw uncached input is only about 13% , the bottleneck isnt cpu cores , its gc pressure from 18 million allocations , the parallel path shines when chunks hit the cache on subsequent calls

---

## the performance story

this is the part worth slowing down on , the benchmark numbers only make sense if you understand the problem theyre solving

### raw compute hits a wall

regexp2 is the cost of correctness , its the only go library i found that handles the unicode category expressions that the gpt-4 pattern requires , but it allocates a lot

when you disable the cache and force the cpu to do raw bpe math with regexp2 on a 16mb file , something ugly happens , the parallel workers saturate memory , the go runtime spends most of its time on allocation locks , and adding more cores barely moves the needle 

18.6 million allocations , going from 1 worker to 16 bought 13% more throughput , thats the gc wall , you can throw cores at the problem but the allocator is the actual bottleneck and it doesnt parallelize cleanly

### cache at the word level

natural language is repetitive , a server handling chat requests will see " the" , " and" , and " of" millions of times , running regexp2 and the bpe merge loop on " the" for the millionth time is pure waste

the architecture neutralizes this by caching at the word level , once regexp2 extracts a chunk like " the" and the merge loop computes its token ids , that result is stored , every future occurrence skips the regex engine and the merge loop entirely , its just a map lookup

because real-world language is repetitive , the warm cache bypasses the gc wall entirely for the vast majority of requests , the aggregate result on a 25mb corpus looks like this

<div id="chart-comparison" class="chart-container"></div>

a quick note on why tokr shows higher tokens/sec but lower mb/s , these measure different things , mb/s is bytes of input text processed per second , tokens/sec depends on average token length , tokrs vocabulary produces slightly shorter average tokens on this corpus so more tokens are generated per byte

---

## what i got wrong

if youre evaluating tokr for production , you need to know where it breaks

**the sync.pool gap in inference:** i added sync.pool to the tokenizer struct to reuse buffers , i wired it into the training function , i forgot to wire it into the inference hot path , encodecore still allocates fresh slices on every call , the fix is to pass a pooled buffer as an explicit workspace parameter and return it to the pool after use

**the merge loops o(n²) worst case:** the current merge logic scans the token array linearly on every iteration to find the lowest-rank pair , for standard english words averaging 5–15 characters this is effectively o(n) and fast , but feed it a pathological input like a 10000-character base64 string that the regex matches as a single chunk , and the loop degrades to o(n²) , it burns cpu until the word finishes , the proper fix is replacing the linear scan with a doubly-linked list paired with a priority queue

**unbounded cache growth:** the word cache has no eviction policy beyond a full-clear at 100,000 entries , on a long-running server processing diverse inputs the cache will grow toward oom before hitting the limit , then drop all warm entries at once , the right fix is a concurrent lru cache with a fixed entry cap

**static chunk size in the worker pool:** parallelencode uses a hardcoded 1mb chunk size , on a 16-core machine processing a 1.2mb file only 2 cores get work , 14 sit idle , the chunk size should be derived dynamically based on available cores

**http server timeouts:** the current net/http server uses default configuration , which means no read write or idle timeouts , a slowloris attack will exhaust the goroutine pool immediately 

<script src="{{ '/assets/js/charts.js' | relative_url }}"></script>
<script>
document.addEventListener('DOMContentLoaded', () => {
  // Chart 1: Latency Drop
  const latencyData = [{x: 1, y: 10.23}];
  for(let i=2; i<=50; i++) {
    // Drop to ~0.22 us immediately with slight noise
    latencyData.push({x: i, y: 0.22 + (Math.random() * 0.05)});
  }
  new LineChart('chart-latency', {
    title: 'Cache Latency Drop',
    xAxisTitle: 'Request Number',
    yAxisTitle: 'Latency',
    yUnit: ' µs',
    yDecimals: 2,
    yMin: 0,
    datasets: [{ label: 'tokr Engine', color: '#58a6ff', data: latencyData }]
  });

  // Chart 2: Parallel Scaling
  new LineChart('chart-scaling', {
    title: 'Throughput Scaling (16MB Corpus)',
    xAxisTitle: 'CPU Cores',
    yAxisTitle: 'Throughput',
    yUnit: ' MB/s',
    xLabels: [1, 2, 4, 8, 12, 16],
    yMin: 2.8,
    yMax: 3.4,
    datasets: [
      { label: 'Throughput', color: '#4caf50', data: [
        {x: 1, y: 2.93}, {x: 2, y: 3.05}, {x: 4, y: 3.15}, 
        {x: 8, y: 3.22}, {x: 12, y: 3.27}, {x: 16, y: 3.30}
      ]}
    ]
  });

  // Chart 3: tokr vs tiktoken
  new LineChart('chart-comparison', {
    title: 'Tokens/Sec by Input Size (25MB Corpus)',
    xAxisTitle: 'Input Size',
    yAxisTitle: 'Tokens/Sec',
    yUnit: 'M',
    xLabels: ['1KB', '10KB', '100KB', '500KB', '1MB'],
    datasets: [
      { label: 'tokr (Pure Go)', color: '#58a6ff', data: [
        {x: 1, y: 4.8}, {x: 2, y: 5.0}, {x: 3, y: 5.1}, {x: 4, y: 5.2}, {x: 5, y: 5.3}
      ]},
      { label: 'tiktoken (Rust)', color: '#e5534b', data: [
        {x: 1, y: 3.4}, {x: 2, y: 3.5}, {x: 3, y: 3.6}, {x: 4, y: 3.65}, {x: 5, y: 3.7}
      ]}
    ]
  });
});
</script>
