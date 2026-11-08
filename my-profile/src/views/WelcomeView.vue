
<script setup>
import { ref } from 'vue'
import { useRouter } from 'vue-router'

const router = useRouter()
const isEntering = ref(false)

const enterPortfolio = () => {
  if (isEntering.value) return

  isEntering.value = true

  setTimeout(() => {
    router.push('/home')
  }, 900)
}
</script>

<template>
  <main
    class="welcome-page relative min-h-screen pt-4 overflow-hidden
           flex items-center justify-center bg-[#030712] px-6"
    :class="{ 'page-exit': isEntering }"
  >

    <!-- Background Glow -->
    <div class="glow glow-one"></div>
    <div class="glow glow-two"></div>
    <div class="glow glow-three"></div>

    <!-- Grid Background -->
    <div class="absolute inset-0 grid-background opacity-30"></div>

    <!-- Decorative Circles -->
    <div class="absolute inset-0 flex items-center justify-center
                pointer-events-none">
      <div class="circle circle-one"></div>
      <div class="circle circle-two"></div>
      <div class="circle circle-three"></div>
    </div>

    <!-- Main Content -->
    <section class="relative z-10 mx-auto max-w-3xl text-center">

      <!-- Top Label -->
      <div class="intro-item mb-8 flex justify-center">
        <div class="inline-flex items-center gap-2 rounded-full
                    border border-blue-500/20 bg-blue-500/10
                    px-5 py-2 backdrop-blur-md">

          <span class="h-2 w-2 rounded-full bg-blue-400
                       animate-pulse"></span>

          <span class="text-xs font-medium tracking-[0.2em]
                       text-blue-300 uppercase">
            Personal Portfolio 2026
          </span>
        </div>
      </div>

      <!-- Profile Initial -->
      <div class="intro-item mb-8 flex justify-center"
           style="animation-delay: 0.15s">

        <div class="profile-logo flex h-24 w-24 items-center
                    justify-center rounded-3xl border
                    border-blue-400/30 bg-blue-500/10
                    shadow-[0_0_60px_rgba(59,130,246,0.2)]">

          <span class="text-4xl font-bold text-white">
            D <span class="text-blue-400"></span>
          </span>
        </div>
      </div>

      <!-- Heading -->
      <div class="intro-item" style="animation-delay: 0.3s">

        <p class="mb-4 text-sm font-medium tracking-[0.3em]
                  text-slate-400 uppercase">
          Hello, Welcome to
        </p>

        <h1 class="text-4xl font-bold tracking-tight
                   text-white sm:text-6xl md:text-7xl">

          Dede Ahmad
          <span class="block mt-2 gradient-text">
            Maolana.
          </span>
        </h1>
      </div>

      <!-- Description -->
      <div class="intro-item mt-7"
           style="animation-delay: 0.45s">

        <p class="mx-auto max-w-2xl text-sm leading-8
                  text-slate-400 sm:text-base">
          Selamat datang di Web pribadi. Jelajahi perjalanan saya, temukan berbagai proyek saya, dan kenali lebih jauh
tentang minat saya pada teknologi dan pengembangan.
        </p>

      </div>

      <!-- Button -->
      <div class="intro-item mt-10"
           style="animation-delay: 0.6s">

        <button
          @click="enterPortfolio"
          :disabled="isEntering"
          class="group relative inline-flex items-center gap-4
                 overflow-hidden rounded-xl bg-blue-600
                 px-8 py-4 font-semibold text-white
                 shadow-[0_10px_40px_rgba(37,99,235,0.25)]
                 transition-all duration-300
                 hover:-translate-y-1 hover:bg-blue-500
                 hover:shadow-[0_15px_50px_rgba(37,99,235,0.4)]
                 disabled:opacity-80"
        >

          <a href="/home" @click.prevent="enterPortfolio">
  Enter My Portfolio
</a>

          <svg
            v-if="!isEntering"
            class="relative z-10 h-5 w-5 transition-transform
                   duration-300 group-hover:translate-x-2"
            fill="none"
            stroke="currentColor"
            viewBox="0 0 24 24"
          >
            <path stroke-linecap="round"
                  stroke-linejoin="round"
                  stroke-width="2"
                  d="M5 12h14m-7-7 7 7-7 7"/>
          </svg>

          <span v-else
                class="relative z-10 h-5 w-5 animate-spin
                       rounded-full border-2 border-white/30
                       border-t-white">
          </span>

          <span class="absolute inset-0 -translate-x-full
                       bg-white/10 transition-transform
                       duration-500 group-hover:translate-x-0">
          </span>

        </button>

      </div>

      <!-- Bottom Text -->
      <div class="intro-item m-2 flex flex-col
                  items-center gap-3"
           style="animation-delay: 0.8s">

        <div class="h-8 w-5 rounded-full border
                    border-slate-500/60 p-1">
          <div class="mx-auto h-2 w-1 rounded-full
                      bg-blue-400 mouse-scroll"></div>
        </div>

        <p class="text-[10px] tracking-[0.3em]
                  text-slate-500 uppercase">
          Discover My World
        </p>

      </div>

    </section>

    <!-- Copyright -->
    <div class="absolute bottom-5 left-0 right-0
                text-center text-xs text-slate-600">
      © 2026 Dede Ahmad Maolana. All rights reserved.
    </div>

    <!-- Transition Overlay -->
    <div class="transition-overlay"
         :class="{ 'overlay-active': isEntering }">
    </div>

  </main>
</template>

<style scoped>

/* Intro Animation */
.intro-item {
  opacity: 0;
  animation: fadeUp 0.9s ease forwards;
}

@keyframes fadeUp {
  from {
    opacity: 0;
    transform: translateY(35px);
    filter: blur(5px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
    filter: blur(0);
  }
}

/* Gradient Text */
.gradient-text {
  background: linear-gradient(
    90deg,
    #60a5fa,
    #818cf8,
    #38bdf8
  );
  background-size: 200% auto;
  background-clip: text;
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  animation: gradientMove 5s linear infinite;
}

@keyframes gradientMove {
  to {
    background-position: 200% center;
  }
}

/* Background Glow */
.glow {
  position: absolute;
  border-radius: 50%;
  filter: blur(100px);
  pointer-events: none;
  animation: glowMove 9s ease-in-out infinite alternate;
}

.glow-one {
  width: 350px;
  height: 350px;
  background: rgba(37, 99, 235, 0.2);
  top: -100px;
  left: -100px;
}

.glow-two {
  width: 400px;
  height: 400px;
  background: rgba(79, 70, 229, 0.15);
  bottom: -150px;
  right: -100px;
  animation-delay: 2s;
}

.glow-three {
  width: 200px;
  height: 200px;
  background: rgba(14, 165, 233, 0.1);
  top: 40%;
  right: 20%;
  animation-delay: 4s;
}

@keyframes glowMove {
  from {
    transform: translate(0, 0) scale(1);
  }
  to {
    transform: translate(50px, 30px) scale(1.2);
  }
}

/* Grid */
.grid-background {
  background-image:
    linear-gradient(rgba(148,163,184,.06) 1px, transparent 1px),
    linear-gradient(90deg, rgba(148,163,184,.06) 1px, transparent 1px);
  background-size: 45px 45px;
  mask-image: linear-gradient(to bottom, black, transparent);
}

/* Circles */
.circle {
  position: absolute;
  border: 1px solid rgba(96, 165, 250, 0.12);
  border-radius: 50%;
  animation: rotateCircle 30s linear infinite;
}

.circle-one {
  width: 500px;
  height: 500px;
}

.circle-two {
  width: 650px;
  height: 650px;
  border-style: dashed;
  animation-direction: reverse;
}

.circle-three {
  width: 800px;
  height: 800px;
  border-color: rgba(96, 165, 250, 0.06);
}

@keyframes rotateCircle {
  to {
    transform: rotate(360deg);
  }
}

/* Profile Logo */
.profile-logo {
  animation: logoFloat 4s ease-in-out infinite;
}

@keyframes logoFloat {
  0%, 100% {
    transform: translateY(0);
  }
  50% {
    transform: translateY(-10px);
  }
}

/* Scroll Indicator */
.mouse-scroll {
  animation: scrollDown 1.5s ease-in-out infinite;
}

@keyframes scrollDown {
  0% {
    opacity: 0;
    transform: translateY(0);
  }
  50% {
    opacity: 1;
  }
  100% {
    opacity: 0;
    transform: translateY(12px);
  }
}

/* Page Transition */
.transition-overlay {
  position: absolute;
  inset: 0;
  z-index: 50;
  background: #030712;
  transform: scaleY(0);
  transform-origin: bottom;
  transition: transform 0.8s cubic-bezier(.76, 0, .24, 1);
  pointer-events: none;
}

.overlay-active {
  transform: scaleY(1);
}

/* Responsive */
@media (max-width: 640px) {
  .circle-one {
    width: 300px;
    height: 300px;
  }

  .circle-two {
    width: 400px;
    height: 400px;
  }

  .circle-three {
    width: 500px;
    height: 500px;
  }
}

/* Accessibility */
@media (prefers-reduced-motion: reduce) {
  .intro-item,
  .glow,
  .circle,
  .profile-logo,
  .gradient-text,
  .mouse-scroll {
    animation: none !important;
    opacity: 1;
  }
}
</style>
