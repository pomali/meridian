<script lang="ts" setup>
const config = useRuntimeConfig();
const { $md } = useNuxtApp();

// Constants
const WORDS_PER_MINUTE = 300;
const HEADER_OFFSET = 80; // Offset for smooth scrolling below fixed headers
const TOC_WIDTH_CLASS = 'w-64'; // Define TOC width here (e.g., w-56, w-64, w-72) - corresponds to 16rem/17rem/18rem
const MAIN_CONTENT_MAX_W_CLASS = 'max-w-3xl'; // Matches your app.vue layout

// Route and brief data extraction (Ensure your getReportBySlug handles async correctly or use useFetch/useAsyncData)
const slug = useRoute().path.split('/').pop()?.replaceAll('_', '/');
if (slug === undefined) {
  throw createError({ statusCode: 404, statusMessage: 'Brief not found' });
}
// Using useAsyncData for potentially async data fetching
const briefData = getReportBySlug(slug);

// Reading progress state
const readingProgress = ref(0);
let scrollListener: () => void;

// Estimate Reading Time
const estimateReadingTime = (content: string): number => {
  if (!content) return 0;
  const wordCount = content.trim().split(/\s+/).length;
  return Math.ceil(wordCount / WORDS_PER_MINUTE);
};

// --- TOC Logic ---
interface TocItem {
  id: string;
  text: string;
  level: number;
} // isUnderlined removed for now
const tocItems = ref<TocItem[]>([]);
const articleContentRef = ref<HTMLElement | null>(null);
const activeHeadingId = ref<string | null>(null);
const mobileMenuOpen = ref(false);
const currentSectionName = ref<string>('on this page');

const generateSlug = (text: string): string => {
  return text
    .toLowerCase()
    .replace(/[^a-z0-9\s-]/g, '')
    .trim()
    .replace(/\s+/g, '-')
    .slice(0, 50);
};

const generateToc = () => {
  if (!articleContentRef.value) return;

  // Query all potential TOC elements: H2, H3, and the specific topic structure
  // IMPORTANT: Adjust 'u > strong' if the LLM output format differs!
  const elements = articleContentRef.value.querySelectorAll('h2, h3, u > strong');
  const newTocItems: TocItem[] = [];

  elements.forEach((el, index) => {
    let level: number;
    let text = el.textContent?.trim() || '';
    let targetElement: HTMLElement = el as HTMLElement; // Default to the element itself

    // Determine level and potentially adjust target element for topics
    if (el.tagName === 'H2') {
      level = 2;
    } else if (el.tagName === 'H3') {
      level = 3;
    } else if (el.tagName === 'STRONG' && el.parentElement?.tagName === 'U') {
      // This is a topic title (<u><strong>...</strong></u>)
      level = 5; // Assign a higher level for topics
      // Target the parent <u> tag for ID and scrolling
      targetElement = el.parentElement;
    } else {
      return; // Skip if it's not a recognized TOC element
    }

    const id = `${level === 5 ? 'topic' : 'section'}-${index}-${generateSlug(text)}`;

    if (text && targetElement) {
      targetElement.id = id; // Assign ID to the H-tag or the <u> tag
      newTocItems.push({ id, text, level });
    }
  });

  tocItems.value = newTocItems; // This flat list is passed to the component
};

// Scroll to section with offset
const scrollToSection = (id: string) => {
  const el = document.getElementById(id);
  if (el) {
    const elementPosition = el.getBoundingClientRect().top;
    const offsetPosition = elementPosition + window.pageYOffset - HEADER_OFFSET;
    window.scrollTo({ top: offsetPosition, behavior: 'smooth' });
  }
};

const scrollToTop = () => {
  window.scrollTo({ top: 0, behavior: 'smooth' });
};

// SEO metadata
const formatDate = computed(() => {
  // Add safety checks for briefData and date
  const date = briefData.value?.date;
  return date && date.month ? `${date.month.toLowerCase()} ${date.day}, ${date.year}` : 'Unknown Date';
});

// Use useSeoMeta for Nuxt 3 SEO
useSeoMeta({
  title: `${briefData.value?.title?.toLowerCase() ?? 'Brief'} | meridian`,
  description: `Intelligence brief for ${formatDate.value}`,
  ogTitle: `${briefData.value?.title ?? 'Intelligence Brief'}`,
  ogDescription: `Intelligence brief for ${formatDate.value}`,
  // Ensure data for OG image exists, provide fallbacks
  ogImage: briefData.value
    ? `${config.public.WORKER_API}/openGraph/brief?title=${encodeURIComponent(briefData.value.title)}&date=${encodeURIComponent(briefData.value.createdAt?.getTime() ?? Date.now())}&articles=${briefData.value.usedArticles ?? 0}&sources=${briefData.value.usedSources ?? 0}`
    : '/default-og-image.png', // Add a default OG image
  ogUrl: `https://news.iliane.xyz/briefs/${slug}`, // Ensure your base URL is correct
  twitterCard: 'summary_large_image',
});

// Add this function near the other TOC-related functions
const updateCurrentSection = (headingId: string | null) => {
  if (!headingId) {
    currentSectionName.value = 'on this page';
    return;
  }

  // Find the current section or topic
  const section = tocItems.value.find(item => {
    if (item.id === headingId) return true;
    // For topics, check if they're under a section
    if (item.level > 3) {
      const parentSection = tocItems.value.find(section => section.level <= 3 && section.id === headingId);
      return parentSection !== undefined;
    }
    return false;
  });

  currentSectionName.value = section ? section.text.toLowerCase() : 'on this page';
};

// Update the watch for activeHeadingId
watch(
  () => activeHeadingId.value,
  newId => {
    updateCurrentSection(newId);
  },
  { immediate: true }
);

// Add these refs with the other state management
const headerRef = ref<HTMLElement | null>(null);
const mobileTocRef = ref<HTMLElement | null>(null);
const isSticky = ref(false);
const showBackToTop = ref(false);

// Add this with the other lifecycle hooks
onMounted(() => {
  // Scroll progress tracking
  scrollListener = () => {
    const scrollTop = document.documentElement.scrollTop;
    const scrollHeight = document.documentElement.scrollHeight - document.documentElement.clientHeight;
    readingProgress.value = scrollHeight > 0 ? (scrollTop / scrollHeight) * 100 : 0;

    // Show back to top button after scrolling down 500px
    showBackToTop.value = scrollTop > 500;
  };
  window.addEventListener('scroll', scrollListener);

  // Generate TOC after initial render ensures elements exist
  nextTick(() => {
    generateToc();
  });

  // Setup intersection observer for sticky behavior
  const observer = new IntersectionObserver(
    ([entry]) => {
      isSticky.value = !entry.isIntersecting;
    },
    { threshold: 0 }
  );

  if (headerRef.value) {
    observer.observe(headerRef.value);
  }

  // Cleanup
  onUnmounted(() => {
    window.removeEventListener('scroll', scrollListener);
    observer.disconnect();
  });
});
</script>

<template>
  <!-- Reading progress bar -->
  <div class="fixed top-0 left-0 w-full h-1 z-50">
    <div
      class="h-full bg-black dark:bg-white transition-all duration-150 ease-out"
      :style="{ width: `${readingProgress}%` }"
    />
  </div>

  <!-- Override the default layout constraints for this page only -->
  <!-- This wrapper pushes its content outside the default max-w-3xl container -->
  <div class="mx-[-1.5rem] w-[calc(100%+3rem)] relative">
    <!-- Inner container to reapply the proper padding -->
    <div class="max-w-3xl mx-auto px-6">
      <!-- Main Content Column -->
      <div>
        <!-- Header section that we'll observe -->
        <header ref="headerRef" class="mb-8">
          <h1 class="text-3xl md:text-4xl font-bold mb-3 leading-tight">
            {{ briefData.title }}
          </h1>
          <div class="flex text-sm text-gray-600 dark:text-gray-400 items-center space-x-2">
            <time>{{ formatDate }}</time>
            <span>•</span>
            <p>{{ estimateReadingTime(briefData.content) }} min read</p>
          </div>
        </header>

        <!-- Mobile TOC (Only visible on small screens) -->
        <div class="xl:hidden mb-8">
          <div
            ref="mobileTocRef"
            :class="[
              'z-40 transition-all duration-200 ', // lower z-index to stay below reading indicator
              isSticky ? 'fixed top-1 left-0 right-0' : 'relative', // top-1 to show reading indicator
            ]"
          >
            <button
              @click="mobileMenuOpen = !mobileMenuOpen"
              class="w-full flex items-center justify-between px-6 py-3 text-sm font-semibold text-gray-700 dark:text-gray-300 bg-white/95 dark:bg-gray-900/95 backdrop-blur-md transition-colors duration-200"
            >
              <span>{{ currentSectionName }}</span>
              <svg
                xmlns="http://www.w3.org/2000/svg"
                viewBox="0 0 20 20"
                fill="currentColor"
                class="w-4 h-4 transform transition-transform duration-300 ease-in-out"
                :class="{ 'rotate-180': mobileMenuOpen }"
              >
                <path
                  fill-rule="evenodd"
                  d="M5.23 7.21a.75.75 0 011.06.02L10 11.168l3.71-3.938a.75.75 0 111.08 1.04l-4.25 4.5a.75.75 0 01-1.08 0l-4.25-4.5a.75.75 0 01.02-1.06z"
                  clip-rule="evenodd"
                />
              </svg>
            </button>
            <Transition
              enter-active-class="transition-all duration-300 ease-out"
              leave-active-class="transition-all duration-200 ease-in"
              enter-from-class="opacity-0 -translate-y-4"
              enter-to-class="opacity-100 translate-y-0"
              leave-from-class="opacity-100 translate-y-0"
              leave-to-class="opacity-0 -translate-y-4"
            >
              <div
                v-if="mobileMenuOpen"
                class="absolute left-0 right-0 bg-white dark:bg-gray-900 shadow-lg p-4 max-h-[60vh] overflow-y-auto"
              >
                <BriefTableOfContents
                  :items="tocItems"
                  :active-heading-id="activeHeadingId"
                  @navigate="
                    id => {
                      scrollToSection(id);
                      mobileMenuOpen = false;
                    }
                  "
                  @active-heading-change="activeHeadingId = $event"
                />
              </div>
            </Transition>
          </div>
        </div>

        <!-- Main article content -->
        <article ref="articleContentRef" class="prose w-full max-w-none" v-html="$md.render(briefData.content)" />

        <!-- Stats and Subscription Form -->
        <div class="mt-16 mb-8">
          <div class="h-px w-full bg-gray-300 dark:bg-gray-700 mb-8" />
          <div class="flex flex-col text-center gap-8">
            <!-- Stat divs remain the same -->
            <div class="grid grid-cols-2 gap-y-6 gap-x-12 text-sm">
              <div>
                <p class="text-gray-600 dark:text-gray-400 mb-1">total articles</p>
                <p class="font-bold text-base">{{ briefData.totalArticles || '-' }}</p>
              </div>
              <div>
                <p class="text-gray-600 dark:text-gray-400 mb-1">total sources</p>
                <p class="font-bold text-base">{{ briefData.totalSources || '-' }}</p>
              </div>
              <div>
                <p class="text-gray-600 dark:text-gray-400 mb-1">used articles</p>
                <p class="font-bold text-base">{{ briefData.usedArticles || '-' }}</p>
              </div>
              <div>
                <p class="text-gray-600 dark:text-gray-400 mb-1">used sources</p>
                <p class="font-bold text-base">{{ briefData.usedSources || '-' }}</p>
              </div>
            </div>

            <div class="text-sm">
              <p class="text-gray-600 dark:text-gray-400 mb-1">final brief generated by</p>
              <p class="font-bold text-base">{{ briefData.model_author }}</p>
            </div>

            <!-- Subscription area -->
            <div class="mt-4 pt-8 border-t border-gray-300 dark:border-gray-700">
              <SubscriptionForm />
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Desktop TOC - Positioned absolutely to the left of the content -->
    <aside
      class="hidden xl:block fixed max-h-[calc(100vh-8rem)] overflow-y-auto pr-8"
      :class="[
        TOC_WIDTH_CLASS,
        // Fixed distance from the left edge at larger screens
        'top-24 right-[78%]',
      ]"
    >
      <ClientOnly>
        <BriefTableOfContents
          :items="tocItems"
          :active-heading-id="activeHeadingId"
          @navigate="scrollToSection"
          @active-heading-change="activeHeadingId = $event"
        />
      </ClientOnly>
    </aside>

    <!-- Back to Top Button -->
    <Transition
      enter-active-class="transition-opacity duration-200"
      leave-active-class="transition-opacity duration-200"
      enter-from-class="opacity-0"
      leave-to-class="opacity-0"
    >
      <button
        v-show="showBackToTop"
        @click="scrollToTop()"
        class="fixed bottom-8 z-50 border border-gray-200 dark:border-gray-950 hover:cursor-pointer right-[max(2rem,calc((100%-68rem)/2))] p-2 rounded-full bg-gray-100 dark:bg-gray-900 text-black dark:text-gray-100 shadow-lg hover:bg-gray-200 dark:hover:bg-gray-800 transition-colors duration-200"
        aria-label="Back to top"
      >
        <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
          <path
            fill-rule="evenodd"
            d="M14.707 12.707a1 1 0 01-1.414 0L10 9.414l-3.293 3.293a1 1 0 01-1.414-1.414l4-4a1 1 0 011.414 0l4 4a1 1 0 010 1.414z"
            clip-rule="evenodd"
          />
        </svg>
      </button>
    </Transition>
  </div>
</template>

<style>
/* Global smooth scrolling */
html {
  scroll-behavior: smooth;
}
</style>

<!-- 
/* Optional: Fine-tune prose dark mode if needed */
.dark .prose {
  /* Example: */
  /* color: theme('colors.slate.300'); */
}
.dark .prose strong {
  /* color: theme('colors.slate.100'); */
}
... etc ... -->
