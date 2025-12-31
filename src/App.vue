<script setup lang="ts">
import { ref, onMounted, computed, watchEffect } from 'vue';
import katex from 'katex';
import 'katex/dist/katex.min.css';

// Type definitions
interface Paper {
  venue: string;
  year: number;
  title: string;
  title_en: string;
  abstract: string;
  abstract_en: string;
  keywords: string;
  authors: string;
  github_links: string;
  pdf_download_url: string;
  openreview_forum_url: string;
  venue_id: string;
}

interface PapersByVenueYear {
  [venue: string]: {
    [year: number]: Paper[];
  };
}

interface PapersData {
  total: number;
  papers_by_venue_year: PapersByVenueYear;
}

// Convert papers data to flat array for easier handling
const allPapers = ref<Paper[]>([]);

// Load papers data from JSON files
const loadPapersData = async () => {
  try {
    // Load first JSON file (converted format with papers_by_venue_year)
    const response1 = await fetch('/paper/data/openreview_zh_converted.json');
    const papersData1 = await response1.json();
    
    // Process first JSON file
    if (papersData1.papers_by_venue_year) {
      const papersDataTyped = papersData1 as unknown as PapersData;
      for (const venue in papersDataTyped.papers_by_venue_year) {
        const venueData = papersDataTyped.papers_by_venue_year[venue];
        if (venueData) {
          for (const yearStr in venueData) {
            const year = parseInt(yearStr);
            const yearPapers = venueData[year];
            if (yearPapers) {
              allPapers.value = [...allPapers.value, ...yearPapers];
            }
          }
        }
      }
    }
    
    // Load second JSON file (array format)
    const response2 = await fetch('/paper/data/openreview_cvpr_aaai.json');
    const papersData2 = await response2.json();
    
    // Process second JSON file
    if (Array.isArray(papersData2)) {
      allPapers.value = [...allPapers.value, ...papersData2];
    }
    
    console.log('Total papers loaded:', allPapers.value.length);
  } catch (error) {
    console.error('Failed to load papers data:', error);
  }
};

// State management
const searchQuery = ref('');
const selectedYear = ref('all');
const selectedVenue = ref('all');
const selectedTags = ref<Set<string>>(new Set());
const excludedTags = ref<Set<string>>(new Set());
const selectionMode = ref(false);
const selectedOnlyMode = ref(false);
const selectedPapers = ref<Set<string>>(new Set());
const scrollProgress = ref(0);
const isChinese = ref(true); // Language toggle state
const showOnlyWithCode = ref(false); // Show only papers with code toggle state - default to false

// Pagination state
const currentPage = ref(1);
const papersPerPage = ref(50); // Default papers per page
const inputPage = ref(''); // For page jump functionality

// Extract all unique years for filter
const years = computed(() => {
  const yearSet = new Set<number>();
  allPapers.value.forEach(paper => yearSet.add(paper.year));
  return Array.from(yearSet).sort((a, b) => b - a);
});

// Extract all unique venues for filter
const venues = computed(() => {
  const venueSet = new Set<string>();
  allPapers.value.forEach(paper => {
    if (paper.venue) {
      venueSet.add(paper.venue);
    }
  });
  return Array.from(venueSet).sort();
});

// Extract all unique tags (keywords) for filter
const tags = computed(() => {
  const tagSet = new Set<string>();
  allPapers.value.forEach(paper => {
    paper.keywords.split(',').forEach(keyword => {
      const trimmedKeyword = keyword.trim();
      if (trimmedKeyword) tagSet.add(trimmedKeyword);
    });
  });
  return Array.from(tagSet).sort();
});

// Enhanced search functionality
const filteredPapers = computed(() => {
  return allPapers.value.filter(paper => {
    // Search filter - enhanced to include more fields
    const query = searchQuery.value.toLowerCase().trim();
    const matchesSearch = !query || (
      paper.title.toLowerCase().includes(query) ||
      paper.authors.toLowerCase().includes(query) ||
      paper.abstract.toLowerCase().includes(query) ||
      paper.keywords.toLowerCase().includes(query) ||
      paper.venue.toLowerCase().includes(query)
    );
    
    // Year filter
    const matchesYear = selectedYear.value === 'all' || paper.year === parseInt(selectedYear.value);
    
    // Venue filter
    const matchesVenue = selectedVenue.value === 'all' || paper.venue === selectedVenue.value;
    
    // Tag filter with improved keyword handling
    // Note: Keywords are separated by semicolons (;) in the data, not commas
    const paperTags = new Set(paper.keywords.split(';').map(k => k.trim().toLowerCase()));
    const processedSelectedTags = new Set(Array.from(selectedTags.value).map(tag => tag.toLowerCase()));
    const processedExcludedTags = new Set(Array.from(excludedTags.value).map(tag => tag.toLowerCase()));
    
    const hasAllIncludedTags = Array.from(processedSelectedTags).every(tag => paperTags.has(tag));
    const hasNoExcludedTags = !Array.from(processedExcludedTags).some(tag => paperTags.has(tag));
    
    // Only show papers with code if showOnlyWithCode is true
    const hasCode = !showOnlyWithCode.value || paper.github_links.trim() !== '';
    
    return matchesSearch && matchesYear && matchesVenue && hasAllIncludedTags && hasNoExcludedTags && hasCode;
  });
});

// Pagination computed properties
const paginatedPapers = computed(() => {
  const startIndex = (currentPage.value - 1) * papersPerPage.value;
  const endIndex = startIndex + papersPerPage.value;
  return filteredPapers.value.slice(startIndex, endIndex);
});

// Total pages
const totalPages = computed(() => {
  return Math.ceil(filteredPapers.value.length / papersPerPage.value);
});

// Enhanced tags with counts for better usability - limited to top 50 most frequent
const tagsWithCounts = computed(() => {
  const tagCountMap = new Map<string, number>();
  
  // Count occurrences of each tag across all papers
  // Note: Keywords are separated by semicolons (;) in the data, not commas
  allPapers.value.forEach(paper => {
    paper.keywords.split(';').forEach(keyword => {
      const trimmedKeyword = keyword.trim().toLowerCase();
      if (trimmedKeyword) {
        tagCountMap.set(trimmedKeyword, (tagCountMap.get(trimmedKeyword) || 0) + 1);
      }
    });
  });
  
  // Convert to array, sort by count (descending), then alphabetically, and limit to top 50
  return Array.from(tagCountMap.entries())
    .map(([tag, count]) => ({ tag, count }))
    .sort((a, b) => {
      // First sort by count descending
      if (b.count !== a.count) {
        return b.count - a.count;
      }
      // Then sort alphabetically
      return a.tag.localeCompare(b.tag);
    })
    .slice(0, 50); // Limit to top 50 most frequent tags
});

// Format tag display with proper capitalization
const formatTag = (tag: string): string => {
  return tag.split(' ')
    .map(word => word.charAt(0).toUpperCase() + word.slice(1).toLowerCase())
    .join(' ');
};

// Render LaTeX in text using KaTeX
const renderLatex = (text: string): string => {
  if (!text) return '';
  
  // Use KaTeX to render LaTeX formulas
  // First handle display math ($$...$$ and \[...\])
  let renderedText = text
    .replace(/\$\$(.*?)\$\$/gs, (match, formula) => {
      try {
        return katex.renderToString(formula, { displayMode: true });
      } catch (e) {
        console.error('Error rendering display math:', e);
        return match;
      }
    })
    .replace(/\\\[(.*?)\\\]/gs, (match, formula) => {
      try {
        return katex.renderToString(formula, { displayMode: true });
      } catch (e) {
        console.error('Error rendering display math:', e);
        return match;
      }
    })
    // Then handle inline math ($...$ and \(...\))
    .replace(/\$(.*?)\$/gs, (match, formula) => {
      try {
        return katex.renderToString(formula, { displayMode: false });
      } catch (e) {
        console.error('Error rendering inline math:', e);
        return match;
      }
    })
    .replace(/\\\((.*?)\\\)/gs, (match, formula) => {
      try {
        return katex.renderToString(formula, { displayMode: false });
      } catch (e) {
        console.error('Error rendering inline math:', e);
        return match;
      }
    });
  
  return renderedText;
};

// Search suggestions based on current query
const searchSuggestions = computed(() => {
  const query = searchQuery.value.toLowerCase().trim();
  if (!query) return [];
  
  const suggestions = new Set<string>();
  
  // Extract suggestions from various fields
  allPapers.value.forEach(paper => {
    // Title suggestions
    if (paper.title.toLowerCase().includes(query)) {
      suggestions.add(paper.title);
    }
    
    // Author suggestions
    paper.authors.split(',').forEach(author => {
      const trimmedAuthor = author.trim().toLowerCase();
      if (trimmedAuthor.includes(query)) {
        suggestions.add(author.trim());
      }
    });
    
    // Keyword suggestions
    // Note: Keywords are separated by semicolons (;) in the data, not commas
    paper.keywords.split(';').forEach(keyword => {
      const trimmedKeyword = keyword.trim().toLowerCase();
      if (trimmedKeyword.includes(query)) {
        suggestions.add(keyword.trim());
      }
    });
  });
  
  return Array.from(suggestions).slice(0, 5); // Limit to top 5 suggestions
});

// Highlight search terms in text
const highlightSearchTerm = (text: string): string => {
  if (!searchQuery.value) return text;
  
  const query = searchQuery.value.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
  const regex = new RegExp(`(${query})`, 'gi');
  return text.replace(regex, '<mark>$1</mark>');
};

// Handle tag filter toggle
const toggleTagFilter = (tag: string) => {
  if (selectedTags.value.has(tag)) {
    selectedTags.value.delete(tag);
    excludedTags.value.add(tag);
  } else if (excludedTags.value.has(tag)) {
    excludedTags.value.delete(tag);
  } else {
    selectedTags.value.add(tag);
  }
};

// Clear search
const clearSearch = () => {
  searchQuery.value = '';
};

// Clear all filters
const clearAllFilters = () => {
  searchQuery.value = '';
  selectedYear.value = 'all';
  selectedVenue.value = 'all';
  selectedTags.value.clear();
  excludedTags.value.clear();
};

// Pagination control methods
// Switch to specified page
const goToPage = (page: number) => {
  if (page >= 1 && page <= totalPages.value) {
    currentPage.value = page;
    window.scrollTo({ top: 0, behavior: 'smooth' });
  }
};

// Go to previous page
const prevPage = () => {
  if (currentPage.value > 1) {
    currentPage.value--;
    window.scrollTo({ top: 0, behavior: 'smooth' });
  }
};

// Go to next page
const nextPage = () => {
  if (currentPage.value < totalPages.value) {
    currentPage.value++;
    window.scrollTo({ top: 0, behavior: 'smooth' });
  }
};

// Change papers per page
const changePapersPerPage = (count: number) => {
  papersPerPage.value = count;
  currentPage.value = 1; // Reset to first page when changing papers per page
  window.scrollTo({ top: 0, behavior: 'smooth' });
};

// Handle page jump
const goToInputPage = () => {
  const pageNumber = parseInt(inputPage.value);
  if (!isNaN(pageNumber) && pageNumber >= 1 && pageNumber <= totalPages.value) {
    currentPage.value = pageNumber;
    inputPage.value = ''; // Clear input field after successful jump
    window.scrollTo({ top: 0, behavior: 'smooth' });
  } else {
    // Clear input field if invalid
    inputPage.value = '';
    alert(`Please enter a valid page number (1-${totalPages.value})`);
  }
};

// Helper methods for pagination display
// Determine if a page button should be shown
const showPageButton = (page: number): boolean => {
  // Always show first, last, and current page
  if (page === 1 || page === totalPages.value || page === currentPage.value) {
    return true;
  }
  
  // Show pages within 3 of current page
  if (page > currentPage.value - 3 && page < currentPage.value + 3) {
    return true;
  }
  
  return false;
};

// Determine if ellipsis should be shown
const showEllipsis = (position: string): boolean => {
  if (totalPages.value <= 7) return false;
  
  if (position === 'left') {
    return currentPage.value > 4;
  } else if (position === 'right') {
    return currentPage.value < totalPages.value - 3;
  }
  
  return false;
};

// Reset page to 1 when filter conditions change
watchEffect(() => {
  // Access filteredPapers to trigger watch when any filter changes
  filteredPapers.value;
  // Reset to first page
  currentPage.value = 1;
});

// Toggle selection mode
const toggleSelectionMode = () => {
  selectionMode.value = !selectionMode.value;
};

// Toggle selected only mode
const toggleSelectedOnly = () => {
  selectedOnlyMode.value = !selectedOnlyMode.value;
};

// Handle paper selection
const togglePaperSelection = (paperTitle: string) => {
  if (selectedPapers.value.has(paperTitle)) {
    selectedPapers.value.delete(paperTitle);
  } else {
    selectedPapers.value.add(paperTitle);
  }
};

// Clear selection
const clearSelection = () => {
  selectedPapers.value.clear();
};

// Scroll functions
const scrollToTop = () => {
  window.scrollTo({ top: 0, behavior: 'smooth' });
};

const scrollToBottom = () => {
  window.scrollTo({ top: document.body.scrollHeight, behavior: 'smooth' });
};

// Update scroll progress
const updateScrollProgress = () => {
  const scrollTop = window.scrollY;
  const scrollHeight = document.documentElement.scrollHeight - document.documentElement.clientHeight;
  const progress = Math.round((scrollTop / scrollHeight) * 100);
  scrollProgress.value = progress;
};

// Abstract toggle state
const expandedAbstracts = ref<Set<string>>(new Set());

const toggleAbstract = (paperTitle: string) => {
  if (expandedAbstracts.value.has(paperTitle)) {
    expandedAbstracts.value.delete(paperTitle);
  } else {
    expandedAbstracts.value.add(paperTitle);
  }
};



// Toggle show only papers with code
const toggleShowOnlyWithCode = () => {
  showOnlyWithCode.value = !showOnlyWithCode.value;
};

// Lifecycle hooks
onMounted(async () => {
  await loadPapersData();
  window.addEventListener('scroll', updateScrollProgress);
  updateScrollProgress();
});
</script>

<template>
  <!-- Floating Navigation -->
  <div class="floating-nav">
    <button @click="scrollToTop" aria-label="Scroll to top">
      <i class="fas fa-arrow-up"></i>
    </button>
    <button @click="scrollToBottom" aria-label="Scroll to bottom">
      <i class="fas fa-arrow-down"></i>
    </button>
    <div class="scroll-progress">{{ scrollProgress }}%</div>
    <!-- Language Toggle Button -->
    <button 
      class="language-toggle" 
      @click="isChinese = !isChinese"
    >
      {{ isChinese ? 'EN' : '中文' }}
    </button>
    <!-- Show Only With Code Toggle Button -->
    <button 
      class="dark-mode-toggle" 
      @click="toggleShowOnlyWithCode"
    >
      <i :class="showOnlyWithCode ? 'fas fa-code' : 'fas fa-code-branch'"></i>
    </button>
  </div>

  <div class="container">





    <!-- Filter Status Bar -->
    <div class="filter-status">
      <div class="preview-header">
        <div class="preview-header-left">
          <span>Active Filters</span>
          <span class="filter-counter">Showing {{ filteredPapers.length }} of {{ allPapers.length }} papers</span>
        </div>
        <div class="preview-header-right">
          <button class="control-button secondary" @click="clearAllFilters">
            <i class="fas fa-trash"></i> Clear All
          </button>
        </div>
      </div>
      <div class="preview-container" id="activeFilters">
        <!-- Search Filter -->
        <div class="filter-tag search" v-if="searchQuery">
          <div class="filter-tag-content">
            <div class="filter-tag-title">Search</div>
            <div class="filter-tag-info">{{ searchQuery }}</div>
          </div>
          <button class="preview-remove" @click="clearSearch" title="Remove search filter">
            <i class="fas fa-times"></i>
          </button>
        </div>
        
        <!-- Year Filter -->
        <div class="filter-tag year" v-if="selectedYear !== 'all'">
          <div class="filter-tag-content">
            <div class="filter-tag-title">Year</div>
            <div class="filter-tag-info">{{ selectedYear }}</div>
          </div>
          <button class="preview-remove" @click="selectedYear = 'all'" title="Remove year filter">
            <i class="fas fa-times"></i>
          </button>
        </div>
        
        <!-- Venue Filter -->
        <div class="filter-tag venue" v-if="selectedVenue !== 'all'">
          <div class="filter-tag-content">
            <div class="filter-tag-title">Venue</div>
            <div class="filter-tag-info">{{ selectedVenue }}</div>
          </div>
          <button class="preview-remove" @click="selectedVenue = 'all'" title="Remove venue filter">
            <i class="fas fa-times"></i>
          </button>
        </div>
        
        <!-- Included Tags Filter -->
        <div 
          class="filter-tag tag include" 
          v-for="tag in selectedTags" 
          :key="'include-' + tag"
        >
          <div class="filter-tag-content">
            <div class="filter-tag-title">Include</div>
            <div class="filter-tag-info">{{ formatTag(tag) }}</div>
          </div>
          <button 
            class="preview-remove" 
            @click="selectedTags.delete(tag)" 
            title="Remove include filter"
          >
            <i class="fas fa-times"></i>
          </button>
        </div>
        
        <!-- Excluded Tags Filter -->
        <div 
          class="filter-tag tag exclude" 
          v-for="tag in excludedTags" 
          :key="'exclude-' + tag"
        >
          <div class="filter-tag-content">
            <div class="filter-tag-title">Exclude</div>
            <div class="filter-tag-info">{{ formatTag(tag) }}</div>
          </div>
          <button 
            class="preview-remove" 
            @click="excludedTags.delete(tag)" 
            title="Remove exclude filter"
          >
            <i class="fas fa-times"></i>
          </button>
        </div>
      </div>
    </div>

    <!-- Enhanced Title with Dataset Statistics -->
    <div class="page-header">
      <div style="display: flex; justify-content: center; align-items: center; margin-bottom: 1.5rem;">
        <h1>Conference Papers Collection</h1>
      </div>
      <div class="header-stats">
        <div class="stat-item">
          <span class="stat-label">Total Papers:</span>
          <span class="stat-value">{{ allPapers.length }}</span>
        </div>
        <div class="stat-item">
          <span class="stat-label">Unique Venues:</span>
          <span class="stat-value">{{ venues.length }}</span>
        </div>
        <div class="stat-item">
          <span class="stat-label">Years Covered:</span>
          <span class="stat-value">{{ years.length }}</span>
        </div>
      </div>
    </div>

    <!-- Filter Info -->
    <div class="filter-info">
      <h3>Filter Options</h3>
      <p><strong>Search:</strong> Enter paper title or author names, then click <i class="fas fa-times"></i> to clear.</p>
      <p><strong>Year:</strong> Filter by publication year</p>
      <p><strong>Tags:</strong> Click once to include (blue), twice to exclude (red), third time to remove filter</p>
      <p><strong>Selection:</strong> Use selection mode to pick and share specific papers</p>
    </div>



    <!-- Filters Bar -->
    <div class="filters">
      <div class="search-wrapper">
        <i class="fas fa-search search-icon"></i>
        <input 
          type="text" 
          id="searchInput" 
          class="search-box" 
          placeholder="Search papers by title, authors, abstract, keywords, venue, or year..." 
          v-model="searchQuery"
          aria-label="Search papers"
        >
        <button class="clear-search-btn" @click="clearSearch" title="Clear search" v-if="searchQuery">
          <i class="fas fa-times"></i>
        </button>
        
        <!-- Search Suggestions Dropdown -->
        <div class="search-suggestions" v-if="searchQuery && searchSuggestions.length > 0">
          <div 
            v-for="(suggestion, index) in searchSuggestions" 
            :key="index"
            class="search-suggestion-item"
            @click="searchQuery = suggestion"
          >
            {{ suggestion }}
          </div>
        </div>
      </div>
      <div class="filter-controls">
        <select id="yearFilter" class="filter-select" v-model="selectedYear">
          <option value="all">All Years</option>
          <option v-for="year in years" :key="year" :value="year">{{ year }}</option>
        </select>
        <select id="venueFilter" class="filter-select" v-model="selectedVenue">
          <option value="all">All Venues</option>
          <option v-for="venue in venues" :key="venue" :value="venue">{{ venue }}</option>
        </select>
      </div>
    </div>

    <!-- Tag Filters -->
    <div class="tag-filters" id="tagFilters">
      <div 
        v-for="{ tag, count } in tagsWithCounts" 
        :key="tag"
        class="tag-filter" 
        :class="{
          'include': selectedTags.has(tag),
          'exclude': excludedTags.has(tag)
        }" 
        :data-tag="tag"
        @click="toggleTagFilter(tag)"
        :title="`${count} papers have this tag`"
      >
        {{ formatTag(tag) }} <span class="tag-count">({{ count }})</span>
      </div>
    </div>

    <!-- Paper Cards -->
    <div class="papers-grid">
      <div 
        v-for="(paper, index) in paginatedPapers" 
        :key="index"
        class="paper-row"
        :data-id="paper.title.toLowerCase().replace(/[^a-z0-9]/g, '')"
        :data-title="paper.title"
        :data-authors="paper.authors"
        :data-year="paper.year"
        :data-tags="paper.keywords.split(',').map(k => k.trim())"
      >
        <div class="paper-card" :class="{ 'selected': selectedPapers.has(paper.title) }">
          <input 
            type="checkbox" 
            class="selection-checkbox" 
            :class="{ 'selection-mode': selectionMode }"
            @click.stop="togglePaperSelection(paper.title)"
          >
          <div class="paper-number">{{ (currentPage - 1) * papersPerPage + index + 1 }}</div>
          <div class="paper-content full-width">
            <h2 class="paper-title">
                <span v-html="renderLatex(isChinese ? paper.title : paper.title_en)"></span>
                <span class="paper-year">({{ paper.year }})</span>
              </h2>
            <div class="paper-meta">
              <span class="paper-venue">{{ paper.venue }}</span>
              <span class="paper-year-label">{{ paper.year }}</span>
            </div>
            <p class="paper-authors">{{ paper.authors }}</p>
            <div class="paper-tags">
              <span 
                v-for="(keyword, idx) in paper.keywords.split(';')" 
                :key="idx"
                class="paper-tag"
              >
                {{ keyword.trim() }}
              </span>
            </div>
            <div class="paper-links">
              <a 
                v-if="paper.pdf_download_url" 
                :href="paper.pdf_download_url" 
                class="paper-link" 
                target="_blank" 
                rel="noopener"
              >
                📄 Paper
              </a>
              <a 
                v-if="paper.openreview_forum_url" 
                :href="paper.openreview_forum_url" 
                class="paper-link" 
                target="_blank" 
                rel="noopener"
              >
                🌐 Forum
              </a>
              <a 
                v-if="paper.github_links" 
                :href="paper.github_links" 
                class="paper-link" 
                target="_blank" 
                rel="noopener"
              >
                💻 Code
              </a>
              <button class="abstract-toggle" @click="toggleAbstract(paper.title)">
                📖 {{ expandedAbstracts.has(paper.title) ? (isChinese ? '隐藏' : 'Hide') : (isChinese ? '显示' : 'Show') }} {{ isChinese ? '摘要' : 'Abstract' }}
              </button>
              <div class="paper-abstract" :class="{ 'show': expandedAbstracts.has(paper.title) }" v-html="renderLatex(isChinese ? paper.abstract : paper.abstract_en)">
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Pagination Controls -->
    <div class="pagination">
      <div class="pagination-info">
        <span>Showing {{ paginatedPapers.length }} of {{ filteredPapers.length }} papers</span>
      </div>
      <div class="pagination-controls">
        <button 
          class="pagination-btn" 
          @click="prevPage" 
          :disabled="currentPage === 1"
        >
          <i class="fas fa-chevron-left"></i> Previous
        </button>
        
        <div class="page-numbers">
          <button 
            v-for="page in totalPages" 
            :key="page"
            class="page-btn"
            :class="{ active: page === currentPage }"
            @click="goToPage(page)"
            v-show="showPageButton(page)"
          >
            {{ page }}
          </button>
          <span v-if="totalPages > 5" class="page-ellipsis" v-show="showEllipsis('left')">...</span>
          <span v-if="totalPages > 5" class="page-ellipsis" v-show="showEllipsis('right')">...</span>
        </div>
        
        <button 
          class="pagination-btn" 
          @click="nextPage" 
          :disabled="currentPage === totalPages"
        >
          Next <i class="fas fa-chevron-right"></i>
        </button>
      </div>
      
      <div class="pagination-per-page">
        <label for="papersPerPage">Show per page:</label>
        <select 
          id="papersPerPage" 
          v-model.number="papersPerPage" 
          @change="changePapersPerPage(papersPerPage)"
        >
          <option :value="10">10</option>
          <option :value="20">20</option>
          <option :value="50">50</option>
          <option :value="100">100</option>
        </select>
      </div>
      
      <div class="pagination-go-to">
        <span>Go to:</span>
        <input 
          type="text" 
          v-model="inputPage" 
          placeholder="Page" 
          class="page-input"
          @keyup.enter="goToInputPage"
        >
        <button class="go-to-btn" @click="goToInputPage">
          Go
        </button>
      </div>
    </div>
  </div>
</template>

<style>
/* Root Variables - Default to Light Mode */
:root {
  --primary-color: #1772d0;
  --hover-color: #f09228;
  --bg-color: #ffffff;
  --card-bg: #ffffff;
  --border-color: #e5e7eb;
  --text-color: #1f2937;
  --search-bg: #ffffff;
  --search-border: #e5e7eb;
  --filter-bg: #ffffff;
  --filter-border: #e5e7eb;
  --tag-bg: #f3f4f6;
  --tag-hover: #e5e7eb;
  --paper-abstract-bg: #f9fafb;
}

/* Base Styles */
body {
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif;
  margin: 0;
  padding: 20px;
  background-color: var(--bg-color);
  color: var(--text-color);
  line-height: 1.5;
  transition: background-color 0.3s, color 0.3s;
}

.container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 0 20px;
  position: relative;
  background-color: var(--bg-color);
  transition: background-color 0.3s;
}

/* Top Right Controls */
.top-right-controls {
  position: absolute;
  top: 20px;
  right: 20px;
  display: flex;
  gap: 0.5rem;
  z-index: 1001;
}

.control-button {
  padding: 0.5rem 1.25rem;
  border: none;
  border-radius: 2rem;
  background-color: var(--primary-color);
  color: white;
  font-size: 0.9rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s ease;
  display: inline-flex;
  align-items: center;
  gap: 0.5rem;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.control-button:hover {
  background-color: #1557b0;
  transform: translateY(-1px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.15);
}

/* Page Header Styling */
.page-header {
  text-align: center;
  margin-bottom: 3rem;
  padding: 2rem;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  border-radius: 1rem;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

.page-header h1 {
  font-size: 2.5rem;
  margin-bottom: 1.5rem;
  color: white;
  font-weight: 700;
  text-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.header-stats {
  display: flex;
  justify-content: center;
  gap: 3rem;
  flex-wrap: wrap;
  margin-top: 1rem;
}

.stat-item {
  text-align: center;
  background: rgba(255, 255, 255, 0.2);
  padding: 1rem 1.5rem;
  border-radius: 0.75rem;
  backdrop-filter: blur(10px);
  min-width: 150px;
  transition: transform 0.2s ease, box-shadow 0.2s ease;
}

.stat-item:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
}

.stat-label {
  display: block;
  font-size: 0.9rem;
  opacity: 0.9;
  margin-bottom: 0.5rem;
  font-weight: 500;
}

.stat-value {
  display: block;
  font-size: 1.8rem;
  font-weight: 700;
  color: white;
}



/* Filter Info */
.filter-info {
  background-color: var(--card-bg);
  border: 1px solid var(--border-color);
  border-radius: 0.5rem;
  padding: 1rem 1.5rem;
  margin-bottom: 2rem;
}

.filter-info h3 {
  margin-top: 0;
  color: var(--primary-color);
  font-size: 1.1rem;
}

.filter-info p {
  margin: 0.5rem 0;
  color: var(--text-color);
}

/* Search and Filters */
.filters {
  display: flex;
  flex-wrap: wrap;
  gap: 1rem;
  margin-bottom: 2rem;
  align-items: center;
  justify-content: space-between;
  background: var(--card-bg);
  padding: 1rem;
  border-radius: 1.25rem;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08);
  border: 1px solid var(--border-color);
}

.filter-controls {
  display: flex;
  gap: 0.75rem;
  align-items: center;
}

/* Ensure filters are responsive */
@media (max-width: 768px) {
  .filters {
    flex-direction: column;
    align-items: stretch;
    gap: 0.75rem;
  }
  
  .filter-controls {
    width: 100%;
    flex-direction: column;
    align-items: stretch;
  }
  
  .filter-select {
    width: 100%;
  }
}

/* Enhanced Search Bar Styling */
.search-wrapper {
  position: relative;
  flex: 1;
  min-width: 250px;
  max-width: 100%;
  display: flex;
  align-items: center;
}

.search-icon {
  position: absolute;
  left: 1rem;
  color: #9ca3af;
  font-size: 1rem;
  z-index: 1;
  transition: color 0.2s ease;
}

.search-box {
  width: 100%;
  padding: 1rem 2.75rem 1rem 2.75rem;
  border: 2px solid var(--border-color);
  border-radius: 1.5rem;
  font-size: 1rem;
  box-sizing: border-box;
  transition: all 0.3s ease;
  background: var(--search-bg);
  color: var(--text-color);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08);
  font-family: inherit;
}

.search-box::placeholder {
  color: #9ca3af;
  opacity: 1;
  transition: opacity 0.2s ease;
}

.search-box:focus {
  outline: none;
  border-color: var(--primary-color);
  box-shadow: 0 0 0 4px rgba(23, 114, 208, 0.15);
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.12);
}

.search-box:focus + .search-icon {
  color: var(--primary-color);
}

.clear-search-btn {
  position: absolute;
  right: 12px;
  top: 50%;
  transform: translateY(-50%);
  background: var(--bg-color);
  border: 2px solid var(--border-color);
  cursor: pointer;
  color: #9ca3af;
  font-size: 0.875rem;
  width: 24px;
  height: 24px;
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1;
  border-radius: 50%;
  transition: all 0.2s ease;
  padding: 0;
}

.clear-search-btn:hover {
  color: #dc2626;
  background: #fee2e2;
  border-color: #dc2626;
  transform: translateY(-50%) scale(1.1);
  box-shadow: 0 2px 4px rgba(220, 38, 38, 0.2);
}

/* Enhanced Filter Select Styling */
.filter-select {
  padding: 0.875rem 2.5rem 0.875rem 1.25rem;
  border: 2px solid var(--border-color);
  border-radius: 1rem;
  background-color: var(--filter-bg);
  color: var(--text-color);
  font-size: 0.95rem;
  cursor: pointer;
  transition: all 0.2s ease;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05);
  appearance: none;
  -webkit-appearance: none;
  background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='16' height='16' viewBox='0 0 24 24' fill='none' stroke='%236b7280' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'%3E%3Cpolyline points='6 9 12 15 18 9'%3E%3C/polyline%3E%3C/svg%3E");
  background-repeat: no-repeat;
  background-position: right 1rem center;
  background-size: 16px;
}

.filter-select:hover {
  border-color: var(--primary-color);
  box-shadow: 0 0 0 3px rgba(23, 114, 208, 0.1);
}

.filter-select:focus {
  outline: none;
  border-color: var(--primary-color);
  box-shadow: 0 0 0 3px rgba(23, 114, 208, 0.1);
  transform: translateY(-1px);
}

/* Search Suggestions */
.search-suggestions {
  position: absolute;
  top: calc(100% + 4px);
  left: 0;
  right: 0;
  background: var(--card-bg);
  border: 1px solid var(--border-color);
  border-radius: 1rem;
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.15);
  z-index: 1001;
  max-height: 250px;
  overflow-y: auto;
  margin: 0;
  list-style: none;
  padding: 0.5rem 0;
  backdrop-filter: blur(10px);
}

/* Custom scrollbar for search suggestions */
.search-suggestions::-webkit-scrollbar {
  width: 6px;
}

.search-suggestions::-webkit-scrollbar-track {
  background: var(--bg-color);
  border-radius: 3px;
}

.search-suggestions::-webkit-scrollbar-thumb {
  background: var(--border-color);
  border-radius: 3px;
}

.search-suggestions::-webkit-scrollbar-thumb:hover {
  background: var(--tag-hover);
}

.search-suggestion-item {
  padding: 0.875rem 1.25rem;
  cursor: pointer;
  transition: all 0.2s ease;
  font-size: 0.95rem;
  color: var(--text-color);
  position: relative;
  overflow: hidden;
}

.search-suggestion-item::before {
  content: '';
  position: absolute;
  left: 0;
  top: 0;
  height: 100%;
  width: 4px;
  background: var(--primary-color);
  transform: scaleY(0);
  transition: transform 0.2s ease;
}

.search-suggestion-item:hover {
  background: var(--tag-hover);
  transform: translateX(4px);
}

.search-suggestion-item:hover::before {
  transform: scaleY(1);
}

.search-suggestion-item:not(:last-child) {
  border-bottom: 1px solid var(--border-color);
}

/* Focus styles for accessibility */
.search-suggestion-item:focus {
  outline: none;
  background: var(--tag-hover);
  transform: translateX(4px);
}

.search-suggestion-item:focus::before {
  transform: scaleY(1);
}

/* Tag Filters */
.tag-filters {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
  margin-bottom: 2rem;
  max-height: 300px;
  overflow-y: auto;
  padding: 0.5rem;
  background: var(--card-bg);
  border-radius: 0.5rem;
  border: 1px solid var(--border-color);
}

.tag-filter {
  background: var(--tag-bg);
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 0.5rem;
  cursor: pointer;
  font-size: 0.9rem;
  transition: all 0.2s;
  color: var(--text-color);
  display: inline-flex;
  align-items: center;
  gap: 0.5rem;
}

.tag-filter:hover {
  background: var(--tag-hover);
  transform: translateY(-1px);
}

.tag-filter.include {
  background: var(--primary-color);
  color: white;
}

.tag-filter.exclude {
  background: #dc2626;
  color: white;
}

/* Tag Count Display */
.tag-count {
  font-size: 0.75rem;
  opacity: 0.8;
  font-weight: 600;
  background: rgba(0, 0, 0, 0.1);
  padding: 0.125rem 0.5rem;
  border-radius: 0.5rem;
}

.tag-filter.include .tag-count,
.tag-filter.exclude .tag-count {
  background: rgba(255, 255, 255, 0.2);
}

/* Marked Search Terms */
mark {
  background: #fef3c7;
  color: #92400e;
  padding: 0.125rem 0.25rem;
  border-radius: 0.25rem;
  font-weight: 500;
}

/* Dark Mode Support for New Elements */
@media (prefers-color-scheme: dark) {
  .search-suggestions {
    background: var(--card-bg);
    border-color: var(--border-color);
  }
  
  .search-suggestion-item {
    color: var(--text-color);
  }
  
  .search-suggestion-item:hover {
    background: #4b5563;
  }
  
  .tag-filters {
    background: var(--card-bg);
  }
  
  mark {
    background: #7c2d12;
    color: #fef3c7;
  }
}

/* Floating Navigation */
.floating-nav {
  position: fixed;
  bottom: 2rem;
  right: 2rem;
  display: flex;
  flex-direction: column;
  gap: 1rem;
  z-index: 1000;
}

.floating-nav button {
  width: 60px;
  height: 60px;
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.9);
  border: 1px solid var(--border-color);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.2);
  transition: all 0.2s;
  font-size: 0.9rem;
  font-weight: 600;
}

.floating-nav button:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
}

/* Language Toggle Button */
.language-toggle {
  background: #1772d0 !important;
  color: white !important;
  border: none !important;
}

.language-toggle:hover {
  background: #1557b0 !important;
}

/* Dark Mode Toggle Button */
.dark-mode-toggle {
  background: #1772d0 !important;
  color: white !important;
  border: none !important;
  font-size: 1.2rem !important;
}

.dark-mode-toggle:hover {
  background: #1557b0 !important;
}

.scroll-progress {
  width: 60px;
  height: 60px;
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.9);
  border: 1px solid var(--border-color);
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 0.9rem;
  font-weight: 600;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.2);
}

/* Selection Preview Bar */
.selection-preview {
  position: sticky;
  top: 0;
  left: 0;
  right: 0;
  width: 100%;
  background: white;
  border-bottom: 1px solid var(--border-color);
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  z-index: 1000;
  display: none;
  margin-bottom: 2rem;
}

.selection-mode .selection-preview {
  display: block;
}

/* Filter Status Bar */
.filter-status {
  position: sticky;
  top: 0;
  left: 0;
  right: 0;
  width: 100%;
  background: var(--card-bg);
  border-bottom: 1px solid var(--border-color);
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  z-index: 999;
  margin-bottom: 2rem;
}

/* Preview Header (shared by both bars) */
.preview-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0.75rem 1rem;
  background: var(--tag-bg);
  border-bottom: 1px solid var(--border-color);
}

.preview-header-left {
  font-weight: 600;
  color: var(--text-color);
  display: flex;
  align-items: center;
  gap: 1rem;
}

.preview-header-right {
  display: flex;
  align-items: center;
  gap: 0.75rem;
}

.selection-counter,
.filter-counter {
  color: #6b7280;
  font-size: 0.9rem;
}

.preview-container {
  padding: 0.75rem 1rem;
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
}

/* Preview Items (shared style for selection and filter tags) */
.preview-item,
.filter-tag {
  flex: 0 0 auto;
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.75rem;
  border: 1px solid var(--border-color);
  border-radius: 0.5rem;
  background: var(--card-bg);
  max-width: 300px;
  margin-bottom: 0.5rem;
}

.preview-content,
.filter-tag-content {
  flex: 1;
  min-width: 0;
  cursor: pointer;
}

.preview-title,
.filter-tag-title {
  font-weight: 600;
  margin-bottom: 0.25rem;
  font-size: 0.9rem;
  color: var(--text-color);
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.preview-authors,
.filter-tag-info {
  font-size: 0.8rem;
  color: #4b5563;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

/* Filter Tag Variations */
.filter-tag.search {
  background: #e6f3ff;
  border-color: #1a73e8;
}

.filter-tag.year {
  background: #e6f7e6;
  border-color: #1e8e1e;
}

.filter-tag.tag {
  background: #f3e6ff;
  border-color: #8e1eb8;
}

/* Remove Button (shared) */
.preview-remove {
  background: none;
  border: none;
  cursor: pointer;
  padding: 2px;
  opacity: 0.7;
  transition: opacity 0.2s;
  display: inline-flex;
  align-items: center;
  justify-content: center;
}

.preview-remove:hover {
  opacity: 1;
}

/* Control Buttons */
.control-button {
  padding: 0.5rem 1rem;
  border-radius: 0.5rem;
  border: none;
  cursor: pointer;
  font-size: 0.9rem;
  display: inline-flex;
  align-items: center;
  gap: 0.5rem;
  transition: all 0.2s;
}

.control-button.primary {
  background-color: var(--primary-color);
  color: white;
}

.control-button.secondary {
  background-color: #f3f4f6;
  color: var(--text-color);
}

.control-button:hover {
  opacity: 0.9;
  transform: translateY(-1px);
}

/* Selection Mode Adjustments */
.selection-mode .floating-nav {
  bottom: 180px;
}

.selection-mode .filter-status {
  top: 84px; /* Height of selection preview */
}

/* Dark Mode Support */
/* For manual light mode toggle */
:not(.dark-mode) {
  --primary-color: #1772d0;
  --hover-color: #f09228;
  --bg-color: #ffffff;
  --card-bg: #ffffff;
  --border-color: #e5e7eb;
  --text-color: #1f2937;
  --search-bg: #ffffff;
  --search-border: #e5e7eb;
  --filter-bg: #ffffff;
  --filter-border: #e5e7eb;
  --tag-bg: #f3f4f6;
  --tag-hover: #e5e7eb;
  --paper-abstract-bg: #f9fafb;
}

.container.dark-mode .paper-card,
.container.dark-mode .donate-box,
.container.dark-mode .filter-info,
.container.dark-mode .tag-filter {
  background: var(--card-bg);
  color: var(--text-color);
  border-color: var(--border-color);
}

.container.dark-mode .paper-tag {
  background: #4b5563;
  color: var(--text-color);
}

/* Apply background color to body based on dark mode state */
body {
  background-color: var(--bg-color);
  color: var(--text-color);
  transition: background-color 0.3s, color 0.3s;
}

.container {
  background-color: var(--bg-color);
  transition: background-color 0.3s;
}

/* Update CSS variables when dark mode is enabled */
.container.dark-mode {
  --primary-color: #1772d0; /* Original blue primary color */
  --hover-color: #1557b0; /* Darker hover color */
  --bg-color: #f8fafc; /* Light background */
  --card-bg: #ffffff; /* White card background */
  --border-color: #e2e8f0; /* Light border color */
  --text-color: #1f2937; /* Dark text color (保持不变) */
  --search-bg: #ffffff; /* White search background */
  --search-border: #e2e8f0; /* Light search border */
  --filter-bg: #ffffff; /* White filter background */
  --filter-border: #e2e8f0; /* Light filter border */
  --tag-bg: #f3f4f6; /* Light tag background */
  --tag-hover: #e5e7eb; /* Tag hover effect */
  --paper-abstract-bg: #f9fafb; /* Light abstract background */
}

/* Remove the old dark mode media query since we now default to dark mode */

/* Print Styles */
@media print {
  .floating-nav,
  .filter-status,
  .selection-preview {
    display: none !important;
  }
}

.selected-only-mode-toggle {
  position: fixed;
  bottom: 6rem;
  right: 2rem;
  background: var(--primary-color);
  color: white;
  padding: 1rem;
  border-radius: 50%;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.2);
  cursor: pointer;
  border: none;
  z-index: 1000;
  width: 60px;
  height: 60px;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: transform 0.2s;
}

.selected-only-mode-toggle:hover {
  transform: scale(1.1);
}

.selected-only-mode-toggle .tooltip {
  position: absolute;
  bottom: 100%;
  right: 0;
  background: rgba(0, 0, 0, 0.8);
  color: white;
  padding: 0.5rem 1rem;
  border-radius: 0.25rem;
  font-size: 0.875rem;
  white-space: nowrap;
  margin-bottom: 0.5rem;
  opacity: 0;
  visibility: hidden;
  transition: opacity 0.2s, visibility 0.2s;
}

.selected-only-mode-toggle:hover .tooltip {
  opacity: 1;
  visibility: visible;
}

.selection-mode-toggle {
  position: fixed;
  bottom: 2rem;
  right: 2rem;
  background: var(--primary-color);
  color: white;
  padding: 1rem;
  border-radius: 50%;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.2);
  cursor: pointer;
  border: none;
  z-index: 1000;
  width: 60px;
  height: 60px;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: transform 0.2s;
}

.selection-mode-toggle:hover {
  transform: scale(1.1);
}

.selection-mode-toggle .tooltip {
  position: absolute;
  bottom: 100%;
  right: 0;
  background: rgba(0, 0, 0, 0.8);
  color: white;
  padding: 0.5rem 1rem;
  border-radius: 0.25rem;
  font-size: 0.875rem;
  white-space: nowrap;
  margin-bottom: 0.5rem;
  opacity: 0;
  visibility: hidden;
  transition: opacity 0.2s, visibility 0.2s;
}

.selection-mode-toggle:hover .tooltip {
  opacity: 1;
  visibility: visible;
}

.selection-checkbox {
  display: none;
  position: absolute;
  top: 1rem;
  right: 1rem;
  width: 2rem;
  height: 2rem;
  z-index: 2;
  cursor: pointer;
  opacity: 1;
  appearance: none;
  -webkit-appearance: none;
  background: white;
  border: 2px solid #e5e7eb;
  border-radius: 50%;
  transition: all 0.2s ease;
}

.selection-checkbox:checked {
  background: #10b981;
  border-color: #10b981;
}

.selection-checkbox:checked::before {
  content: '✓';
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  color: white;
  font-size: 1.2rem;
}

.selection-mode .selection-checkbox {
  display: block !important;
  pointer-events: auto !important;
}

/* Paper Cards */
.papers-grid {
  display: grid;
  gap: 2rem;
}

.paper-card {
  background: var(--card-bg);
  border: 1px solid var(--border-color);
  border-radius: 0.75rem;
  padding: 2rem;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
  transition: transform 0.2s ease, box-shadow 0.2s ease;
  display: flex;
  gap: 1.5rem;
  position: relative;
}

.paper-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

.paper-card.selected {
  border: 2px solid var(--primary-color);
  box-shadow: 0 0 0 1px var(--primary-color);
}

.paper-number {
  position: absolute;
  top: -1rem;
  left: -1rem;
  background-color: var(--primary-color);
  color: white;
  width: 2rem;
  height: 2rem;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: bold;
  z-index: 1;
}

.paper-content.full-width {
  flex: 1 1 100%;
  width: 100%;
}

.paper-content {
  flex: 1;
  min-width: 0;
}

.paper-title {
  font-size: 1.25rem;
  font-weight: 600;
  margin: 0 0 1rem 0;
  color: var(--text-color);
}

.paper-authors {
    color: #4b5563;
    margin-bottom: 1rem;
}

.paper-meta {
    display: flex;
    align-items: center;
    gap: 1rem;
    margin-bottom: 0.75rem;
    font-size: 0.9rem;
    color: var(--primary-color);
}

.paper-venue {
    background: rgba(23, 114, 208, 0.1);
    padding: 0.25rem 0.75rem;
    border-radius: 0.5rem;
    font-weight: 600;
    color: var(--primary-color);
}

.paper-year-label {
    background: rgba(16, 185, 129, 0.1);
    padding: 0.25rem 0.75rem;
    border-radius: 0.5rem;
    font-weight: 600;
    color: #10b981;
}

.paper-tags {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
  margin-bottom: 1rem;
}

.paper-tag {
  background: var(--tag-bg);
  padding: 0.25rem 0.75rem;
  border-radius: 0.5rem;
  font-size: 0.85rem;
  color: var(--text-color);
}

.paper-links {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
  margin-top: 1rem;
  align-items: center;
}

.paper-link, .abstract-toggle {
  color: var(--primary-color);
  text-decoration: none;
  display: inline-flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.5rem 1rem;
  background: var(--tag-bg);
  border: none;
  border-radius: 0.5rem;
  transition: all 0.2s;
  font-size: 0.9rem;
  cursor: pointer;
  white-space: nowrap;
}

.paper-link:hover, .abstract-toggle:hover {
  background: var(--tag-hover);
  color: var(--hover-color);
}

.paper-abstract {
  margin-top: 1rem;
  display: none;
  background: var(--paper-abstract-bg);
  padding: 1rem;
  border-radius: 0.5rem;
  color: var(--text-color);
  line-height: 1.6;
  border: 1px solid var(--border-color);
}

.paper-abstract.show {
  display: block;
}

/* Responsive Design */
@media (max-width: 1024px) {
  .container {
    padding: 0 1rem;
  }
  
  .selection-preview {
    display: none !important;
  }
}

@media (max-width: 768px) {
  body {
    padding: 10px;
  }
  
  .container {
    padding: 0 1rem;
  }
  
  .filters {
    flex-direction: column;
    align-items: stretch;
  }
  
  .search-wrapper {
    width: 100%;
  }
  
  .paper-card {
    flex-direction: column;
  }
  
  .paper-thumbnail {
    width: 100%;
    height: 200px;
    flex: none;
    aspect-ratio: 1.2/1;
  }
  
  .floating-nav {
    bottom: 140px;
    right: 1rem;
  }
  
  .floating-nav button,
  .scroll-progress {
    width: 40px;
    height: 40px;
  }
  
  .bitcoin-info {
    flex-direction: column;
  }
  
  .selection-controls {
    flex-direction: column;
    align-items: stretch;
  }
  
  .paper-links {
    flex-direction: column;
  }
  
  .share-modal-content {
    padding: 1rem;
    width: 95%;
  }
  
  .share-url-container {
    flex-direction: column;
  }
}

@media (max-width: 480px) {
  h1 {
    font-size: 1.8rem;
  }
  
  .paper-card {
    padding: 1rem;
  }
  
  .tag-filters {
    gap: 0.25rem;
  }
  
  .tag-filter {
    font-size: 0.8rem;
    padding: 0.25rem 0.5rem;
  }
}

/* Pagination Styles */
.pagination {
  display: flex;
  flex-wrap: wrap;
  justify-content: space-between;
  align-items: center;
  margin: 2rem 0;
  padding: 1rem;
  background: var(--card-bg);
  border-radius: 0.75rem;
  border: 1px solid var(--border-color);
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

.pagination-info {
  font-size: 0.95rem;
  color: var(--text-color);
  margin-bottom: 1rem;
  flex-basis: 100%;
  text-align: center;
}

.pagination-controls {
  display: flex;
  align-items: center;
  gap: 1rem;
  margin-bottom: 1rem;
}

.pagination-btn {
  padding: 0.75rem 1.25rem;
  border: 1px solid var(--border-color);
  border-radius: 0.5rem;
  background: var(--card-bg);
  color: var(--text-color);
  cursor: pointer;
  transition: all 0.2s ease;
  font-size: 0.9rem;
  font-weight: 500;
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.pagination-btn:hover:not(:disabled) {
  background: var(--tag-bg);
  border-color: var(--primary-color);
  transform: translateY(-1px);
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.pagination-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
  transform: none;
}

.page-numbers {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.page-btn {
  width: 2.5rem;
  height: 2.5rem;
  border: 1px solid var(--border-color);
  border-radius: 0.5rem;
  background: var(--card-bg);
  color: var(--text-color);
  cursor: pointer;
  transition: all 0.2s ease;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 500;
  font-size: 0.9rem;
}

.page-btn:hover {
  border-color: var(--primary-color);
  background: var(--tag-bg);
  transform: translateY(-1px);
}

.page-btn.active {
  background: var(--primary-color);
  color: white;
  border-color: var(--primary-color);
  box-shadow: 0 2px 4px rgba(23, 114, 208, 0.2);
}

.page-ellipsis {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 2.5rem;
  height: 2.5rem;
  font-size: 1.2rem;
  color: var(--text-color);
  opacity: 0.7;
}

.pagination-per-page {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  font-size: 0.9rem;
  color: var(--text-color);
}

.pagination-per-page select {
  padding: 0.5rem 1rem;
  border: 1px solid var(--border-color);
  border-radius: 0.5rem;
  background: var(--card-bg);
  color: var(--text-color);
  font-size: 0.9rem;
  cursor: pointer;
  transition: all 0.2s ease;
}

.pagination-per-page select:hover {
  border-color: var(--primary-color);
}

/* Page Jump Styles */
.pagination-go-to {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  font-size: 0.9rem;
  color: var(--text-color);
  margin-top: 1rem;
}

@media (min-width: 768px) {
  .pagination-go-to {
    margin-top: 0;
  }
}

.page-input {
  padding: 0.5rem 1rem;
  border: 1px solid var(--border-color);
  border-radius: 0.5rem;
  background: var(--card-bg);
  color: var(--text-color);
  font-size: 0.9rem;
  width: 60px;
  text-align: center;
  transition: all 0.2s ease;
}

.page-input:focus {
  outline: none;
  border-color: var(--primary-color);
  box-shadow: 0 0 0 2px rgba(23, 114, 208, 0.2);
}

.go-to-btn {
  padding: 0.5rem 1rem;
  border: 1px solid var(--primary-color);
  border-radius: 0.5rem;
  background: var(--primary-color);
  color: white;
  font-size: 0.9rem;
  cursor: pointer;
  transition: all 0.2s ease;
  font-weight: 500;
}

.go-to-btn:hover {
  background: #1557b0;
  border-color: #1557b0;
  transform: translateY(-1px);
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

/* Responsive Pagination */
@media (min-width: 768px) {
  .pagination-info {
    flex-basis: auto;
    text-align: left;
    margin-bottom: 0;
  }
  
  .pagination-controls {
    margin-bottom: 0;
  }
}

@media (max-width: 767px) {
  .pagination {
    flex-direction: column;
    align-items: center;
    gap: 1rem;
  }
  
  .pagination-controls {
    flex-wrap: wrap;
    justify-content: center;
  }
  
  .pagination-per-page {
    width: 100%;
    justify-content: center;
  }
  
  .page-btn {
    width: 2rem;
    height: 2rem;
    font-size: 0.8rem;
  }
  
  .pagination-btn {
    padding: 0.5rem 1rem;
    font-size: 0.8rem;
  }
}

/* Page Jump Styles */
.pagination-go-to {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  font-size: 0.9rem;
  color: var(--text-color);
  margin-top: 1rem;
  flex-basis: 100%;
  justify-content: center;
}

.pagination-go-to span {
  font-weight: 500;
}

.page-input {
  width: 5rem;
  padding: 0.5rem 0.75rem;
  border: 1px solid var(--border-color);
  border-radius: 0.5rem;
  background: var(--card-bg);
  color: var(--text-color);
  font-size: 0.9rem;
  font-family: inherit;
  text-align: center;
  transition: all 0.2s ease;
}

.page-input:focus {
  outline: none;
  border-color: var(--primary-color);
  box-shadow: 0 0 0 3px rgba(23, 114, 208, 0.1);
}

.go-to-btn {
  padding: 0.5rem 1.25rem;
  border: 1px solid var(--border-color);
  border-radius: 0.5rem;
  background: var(--card-bg);
  color: var(--text-color);
  cursor: pointer;
  transition: all 0.2s ease;
  font-size: 0.9rem;
  font-weight: 500;
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.go-to-btn:hover {
  background: var(--tag-bg);
  border-color: var(--primary-color);
  transform: translateY(-1px);
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.go-to-btn:active {
  transform: translateY(0);
}

/* Responsive for Page Jump */
@media (min-width: 768px) {
  .pagination-go-to {
    flex-basis: auto;
    margin-top: 0;
    justify-content: flex-end;
  }
}
</style>
