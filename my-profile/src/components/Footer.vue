
<script setup>
import { computed } from 'vue'

const currentYear = new Date().getFullYear()

const quickLinks = [
  { name: 'About Me', href: '/About' },
  { name: 'Portofolio', href: '/Portofolio' },
  { name: 'Contact', href: '/Contact' }
]

const socialLinks = [
  {
    name: 'GitHub',
    url: 'https://github.com/dee-aah',
    icon: 'github'
  },
  {
    name: 'LinkedIn',
    url: 'https://www.linkedin.com/in/dede-ahmad-m-54834b302?utm_source=share_via&utm_content=profile&utm_medium=member_android',
    icon: 'linkedin'
  },
  {
    name: 'Instagram',
    url: 'https://www.instagram.com/dede_am4?utm_source=qr&igsh=MXR5NXh2cWtvYXNrZA==',
    icon: 'instagram'
  }
]
</script>

<template>
  <footer class="relative overflow-hidden bg-slate-950 text-slate-300">

    <!-- Decorative background -->
    <div class="absolute -top-32 left-1/2 h-64 w-64
                -translate-x-1/2 rounded-full
                bg-blue-600/10 blur-3xl">
    </div>

    <div class="relative mx-auto max-w-7xl px-6 pt-16 pb-8">

      <!-- Main Footer -->
      <div class="grid grid-cols-1 gap-12 md:grid-cols-2 lg:grid-cols-4">

        <!-- Brand -->
        <div class="lg:col-span-2">
          <a href="#home"
             class="text-3xl font-bold tracking-tight text-white">
            Dede Ahmad Maolana<span class="text-blue-500">.</span>
          </a>

          <p class="mt-5 max-w-md text-sm text-justify leading-7 text-slate-400">
            Saya seorang developer yang memiliki ketertarikan dalam pengembangan website dan teknologi. 
            Saya senang membangun solusi digital yang modern, responsif, dan bermanfaat.
          </p>

          <div class="mt-6 flex items-center gap-3">
            <span class="relative flex h-3 w-3">
              <span class="absolute inline-flex h-full w-full
                           animate-ping rounded-full bg-green-400 opacity-75">
              </span>
              <span class="relative inline-flex h-3 w-3
                           rounded-full bg-green-500">
              </span>
            </span>

            <span class="text-xs text-slate-400">
              Available for Freelance
            </span>
          </div>
        </div>

        <!-- Navigation -->
        <div>
          <h3 class="mb-6 text-sm font-semibold uppercase
                     tracking-wider text-white">
            Quick Links
          </h3>

          <ul class="space-y-4">
            <li v-for="link in quickLinks" :key="link.name">
              <a :href="link.href"
                 class="text-sm text-slate-400 transition
                        duration-300 hover:translate-x-1
                        hover:text-blue-400 inline-block">
                {{ link.name }}
              </a>
            </li>
          </ul>
        </div>

        <!-- Social Media -->
        <div>
          <h3 class="mb-6 text-sm font-semibold uppercase
                     tracking-wider text-white">
            Connect With Me
          </h3>

          <div class="flex flex-col gap-4">
            <a
              v-for="social in socialLinks"
              :key="social.name"
              :href="social.url"
              target="_blank"
              rel="noopener noreferrer"
              class="group flex items-center gap-3 text-sm
                     text-slate-400 transition hover:text-blue-400"
            >
              <span class="flex h-9 w-9 items-center justify-center
                           rounded-lg border border-slate-800
                           bg-slate-900 transition
                           group-hover:border-blue-500
                           group-hover:bg-blue-500/10">
                <svg v-if="social.icon === 'github'"
                     class="h-4 w-4" viewBox="0 0 24 24"
                     fill="currentColor">
                  <path d="M12 .5a12 12 0 0 0-3.79 23.39c.6.11.82-.26.82-.58v-2.05c-3.34.73-4.04-1.42-4.04-1.42-.55-1.39-1.33-1.76-1.33-1.76-1.09-.75.08-.74.08-.74 1.2.09 1.83 1.23 1.83 1.23 1.07 1.83 2.8 1.3 3.49.99.11-.78.42-1.3.76-1.6-2.67-.3-5.47-1.34-5.47-5.95 0-1.31.47-2.38 1.23-3.22-.12-.3-.53-1.52.12-3.17 0 0 1-.32 3.3 1.23a11.5 11.5 0 0 1 6 0c2.29-1.55 3.29-1.23 3.29-1.23.65 1.65.24 2.87.12 3.17.77.84 1.23 1.91 1.23 3.22 0 4.62-2.8 5.64-5.48 5.94.43.37.81 1.1.81 2.22v3.3c0 .32.22.69.83.57A12 12 0 0 0 12 .5Z"/>
                </svg>

                <svg v-else-if="social.icon === 'linkedin'"
                     class="h-4 w-4" viewBox="0 0 24 24"
                     fill="currentColor">
                  <path d="M20.45 20.45h-3.56v-5.57c0-1.33-.03-3.04-1.85-3.04-1.85 0-2.14 1.45-2.14 2.94v5.67H9.34V9h3.42v1.56h.05c.48-.9 1.64-1.85 3.37-1.85 3.6 0 4.27 2.37 4.27 5.46v6.28ZM5.32 7.43a2.06 2.06 0 1 1 0-4.12 2.06 2.06 0 0 1 0 4.12ZM7.1 20.45H3.54V9H7.1v11.45ZM22.23 0H1.77C.79 0 0 .77 0 1.73v20.54C0 23.23.79 24 1.77 24h20.46c.98 0 1.77-.77 1.77-1.73V1.73C24 .77 23.21 0 22.23 0Z"/>
                </svg>

                <svg v-else
                     class="h-4 w-4" viewBox="0 0 24 24"
                     fill="currentColor">
                  <path d="M12 2.16c3.2 0 3.58.01 4.85.07 3.25.15 4.77 1.69 4.92 4.92.06 1.27.07 1.65.07 4.85s-.01 3.58-.07 4.85c-.15 3.22-1.67 4.77-4.92 4.92-1.27.06-1.65.07-4.85.07s-3.58-.01-4.85-.07c-3.24-.15-4.77-1.7-4.92-4.92-.06-1.27-.07-1.65-.07-4.85s.01-3.58.07-4.85c.15-3.23 1.68-4.77 4.92-4.92 1.27-.06 1.65-.07 4.85-.07ZM12 0C8.75 0 8.34.01 7.05.07 2.8.27.27 2.79.07 7.05.01 8.34 0 8.75 0 12s.01 3.66.07 4.95c.2 4.25 2.72 6.78 6.98 6.98C8.34 23.99 8.75 24 12 24s3.66-.01 4.95-.07c4.25-.2 6.78-2.72 6.98-6.98.06-1.29.07-1.7.07-4.95s-.01-3.66-.07-4.95C23.73 2.8 21.21.27 16.95.07 15.66.01 15.25 0 12 0Zm0 5.84A6.16 6.16 0 1 0 12 18.16 6.16 6.16 0 0 0 12 5.84Zm0 10.16a4 4 0 1 1 0-8 4 4 0 0 1 0 8Zm6.41-11.85a1.44 1.44 0 1 0 0 2.88 1.44 1.44 0 0 0 0-2.88Z"/>
                </svg>
              </span>

              {{ social.name }}
            </a>
          </div>
        </div>
      </div>

      <!-- Divider -->
      <div class="my-10 border-t border-slate-800"></div>

      <!-- Bottom Footer -->
      <div class="flex flex-col items-center justify-between
                  gap-4 text-center sm:flex-row sm:text-left">

        <p class="text-xs text-slate-500">
          © {{ currentYear }} Dede Ahmad Maolana.
          All rights reserved.
        </p>

        <p class="text-xs text-slate-500">
          Designed & Built with
          using Vue.js & Tailwind CSS
        </p>
      </div>
    </div>
  </footer>
</template>
