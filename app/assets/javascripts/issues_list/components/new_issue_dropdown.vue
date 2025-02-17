<script>
import {
  GlDropdown,
  GlDropdownItem,
  GlDropdownText,
  GlLoadingIcon,
  GlSearchBoxByType,
} from '@gitlab/ui';
import createFlash from '~/flash';
import searchProjectsQuery from '~/issues_list/queries/search_projects.query.graphql';
import { DASH_SCOPE, joinPaths } from '~/lib/utils/url_utility';
import { __, sprintf } from '~/locale';
import { DEBOUNCE_DELAY } from '~/vue_shared/components/filtered_search_bar/constants';

export default {
  i18n: {
    defaultDropdownText: __('Select project to create issue'),
    noMatchesFound: __('No matches found'),
    toggleButtonLabel: __('Toggle project select'),
  },
  components: {
    GlDropdown,
    GlDropdownItem,
    GlDropdownText,
    GlLoadingIcon,
    GlSearchBoxByType,
  },
  inject: ['fullPath'],
  data() {
    return {
      projects: [],
      search: '',
      selectedProject: {},
      shouldSkipQuery: true,
    };
  },
  apollo: {
    projects: {
      query: searchProjectsQuery,
      variables() {
        return {
          fullPath: this.fullPath,
          search: this.search,
        };
      },
      update: ({ group }) => group.projects.nodes ?? [],
      error(error) {
        createFlash({
          message: __('An error occurred while loading projects.'),
          captureError: true,
          error,
        });
      },
      skip() {
        return this.shouldSkipQuery;
      },
      debounce: DEBOUNCE_DELAY,
    },
  },
  computed: {
    dropdownHref() {
      return this.hasSelectedProject
        ? joinPaths(this.selectedProject.webUrl, DASH_SCOPE, 'issues/new')
        : undefined;
    },
    dropdownText() {
      return this.hasSelectedProject
        ? sprintf(__('New issue in %{project}'), { project: this.selectedProject.name })
        : this.$options.i18n.defaultDropdownText;
    },
    hasSelectedProject() {
      return this.selectedProject.id;
    },
    showNoSearchResultsText() {
      return !this.projects.length && this.search;
    },
  },
  methods: {
    handleDropdownClick() {
      if (!this.dropdownHref) {
        this.$refs.dropdown.show();
      }
    },
    handleDropdownShown() {
      if (this.shouldSkipQuery) {
        this.shouldSkipQuery = false;
      }
      this.$refs.search.focusInput();
    },
    selectProject(project) {
      this.selectedProject = project;
    },
  },
};
</script>

<template>
  <gl-dropdown
    ref="dropdown"
    right
    split
    :split-href="dropdownHref"
    :text="dropdownText"
    :toggle-text="$options.i18n.toggleButtonLabel"
    variant="confirm"
    @click="handleDropdownClick"
    @shown="handleDropdownShown"
  >
    <gl-search-box-by-type ref="search" v-model.trim="search" />
    <gl-loading-icon v-if="$apollo.queries.projects.loading" />
    <template v-else>
      <gl-dropdown-item
        v-for="project of projects"
        :key="project.id"
        @click="selectProject(project)"
      >
        {{ project.nameWithNamespace }}
      </gl-dropdown-item>
      <gl-dropdown-text v-if="showNoSearchResultsText">
        {{ $options.i18n.noMatchesFound }}
      </gl-dropdown-text>
    </template>
  </gl-dropdown>
</template>
