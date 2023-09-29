---
info: Slides for presentation at Functional Scala 2023
theme: 'default'
presenter: false
download: false
exportFilename: 'presentation'
export:
  format: pdf
  timeout: 30000
  dark: auto
  withClicks: true
  withToc: true
highlighter: 'shiki'
lineNumbers: true
monaco: false
remoteAssets: true
selectable: true
record: false
colorSchema: 'dark'
aspectRatio: '16/9'
canvasWidth: 980
favicon: 'functionalScalaIcon.png'
drawings:
  enabled: false
  persist: false
  presenterOnly: false
  syncAll: true

transition: slide-left
background: /conquer.jpg
class: 'text-center pt-0 pb-23'
---

### Elevate and Conquer: Unleashing the Power of High-Level Endpoints in Scala 

<b>December 2023</b>

<div class="absolute top-10 right-16">
  <img src="/functionalScalaLogo.png" class="h-10" />
</div>

---
transition: slide-left
layout: image-right
image: /yo.jpg
class: "flex items-center text-2xl"
---

<div>
  <p>Jorge Vásquez</p>
  <p>Scala Developer</p>
  <p class="text-[#f2644f]">@Ziverge</p>
</div>


---
transition: slide-left
layout: image
image: /laptop.jpg
class: "flex h-screen justify-end items-center"
---

# Introduction

<style>
h1 {
  @apply text-6xl !important
}
</style>

---
transition: slide-left
layout: image
image: /laptop.jpg
class: "flex h-screen justify-end items-center text-right"
---

# Problem: <br/>Launching a **Web Service**

<style>
h1 {
  @apply text-3xl !important
}
</style>

---
transition: slide-left
layout: quote
class: "flex h-screen justify-center items-center"
---

# Can we do **better?**

<style>
h1 {
  @apply text-6xl !important
}
</style>

---
transition: slide-left
layout: image
image: /theater.jpg
class: "flex h-screen justify-center items-center"
---

# Use Endpoints!

<style>
h1 {
  @apply text-6xl !important
}
</style>

---
transition: slide-left
layout: image-right
image: /summary.jpg
---

# **Summary**

<div class="flex w-full h-4/5 items-center gap-5">
  <div>
    <ul>
      <li v-click>Classic approach to building APIs is low-level</li>
      <li v-click>Endpoints offer a higher-level solution</li>
      <li v-click>Each endpoint library has slightly different features and excels at slightly different use cases</li>
    </ul>
  </div>
</div>

---
transition: slide-left
layout: image-right
image: /learn.jpg
---

# **To learn more...**

<div class="flex w-full h-3/5 items-center gap-5">
  <div>
    <ul>
      <li>Visit the <strong>Tapir</strong> <a href="https://github.com/softwaremill/tapir" target="_blank">GitHub repo</a></li>
      <li>Visit the <strong>Endpoints4s</strong> <a href="https://github.com/endpoints4s/endpoints4s" target="_blank">GitHub repo</a></li>
      <li>Visit the <strong>ZIO HTTP</strong> <a href="https://github.com/zio/zio-http" target="_blank">GitHub repo</a></li>
    </ul>
  </div>
</div>

---
transition: slide-left
layout: image-right
image: /thanks.jpg
---

# **Special Thanks**

<div class="flex w-full h-3/5 items-center gap-5">
  <div>
    <ul>
      <li><strong>Ziverge</strong> for organizing this conference</li>
      <li><strong>John De Goes</strong> for guidance and support</li>
    </ul>
  </div>
</div>

---
transition: slide-left
layout: image-right
image: /computer.png
---

# **Contact me**

<div class="grid grid-cols-8 gap-4 items-center h-4/5 content-center text-2xl">
  <div class="col-span-1"><img src="/x.png" class="w-8" /></div> <div class="col-span-7">@jorvasquez2301</div>
  <div class="col-span-1"><img src="/linkedin.png" class="w-8" /></div> <div class="col-span-7">jorge-vasquez-2301</div>
  <div class="col-span-1"><img src="/email.png" class="w-8" /></div> <div class="col-span-7">jorge.vasquez@ziverge.com</div>
</div>