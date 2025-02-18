<template lang="pug">
div
  h5
    icon.handle(name="bars" v-if="editable" style="opacity: 0.6; cursor: grab;")
    | {{ visualizations[type].title }}
  div(v-if="editable").vis-style-dropdown-btn
    b-dropdown.mr-1(size="sm" variant="outline-secondary" right)
      template(v-slot:button-content)
        icon(name="cog")
      b-dropdown-item(v-for="t in types" :key="t" variant="outline-secondary" @click="$emit('onTypeChange', id, t)")
        | {{ visualizations[t].title }} #[span.small(v-if="!visualizations[t].available" style="color: #A50") (no data)]
    b-button.p-0(size="sm", variant="outline-danger" @click="$emit('onRemove', id)")
      icon(name="times")

  div(v-if="!supports_period")
    b-alert.small.px-2.py-1(show variant="warning")
      | This feature doesn't support the current time period.

  div(v-if="$store.state.activity.buckets.loaded")
    // Check data prerequisites
    div(v-if="!has_prerequisites")
      b-alert.small.px-2.py-1(show variant="warning")
        | This feature is missing data from a required watcher.
        | You can find a list of all watchers in #[a(href="https://activitywatch.readthedocs.io/en/latest/watchers.html") the documentation].

    div(v-if="type == 'top_apps'")
      aw-summary(:fields="$store.state.activity.window.top_apps",
                 :namefunc="e => e.data.app",
                 :colorfunc="e => e.data.app",
                 with_limit)
    div(v-if="type == 'top_titles'")
      aw-summary(:fields="$store.state.activity.window.top_titles",
                 :namefunc="e => e.data.title",
                 :colorfunc="e => e.data.title",
                 with_limit)
    div(v-if="type == 'top_domains'")
      aw-summary(:fields="$store.state.activity.browser.top_domains",
                 :namefunc="e => e.data.$domain",
                 :colorfunc="e => e.data.$domain",
                 with_limit)
    div(v-if="type == 'top_urls'")
      aw-summary(:fields="$store.state.activity.browser.top_urls",
                 :namefunc="e => e.data.url",
                 :colorfunc="e => e.data.$domain",
                 with_limit)
    div(v-if="type == 'top_editor_files'")
      aw-summary(:fields="$store.state.activity.editor.top_files",
                 :namefunc="top_editor_files_namefunc",
                 :hoverfunc="top_editor_files_hoverfunc",
                 :colorfunc="e => e.data.language",
                 with_limit)
    div(v-if="type == 'top_editor_languages'")
      aw-summary(:fields="$store.state.activity.editor.top_languages",
                 :namefunc="e => e.data.language",
                 :colorfunc="e => e.data.language",
                 with_limit)
    div(v-if="type == 'top_editor_projects'")
      aw-summary(:fields="$store.state.activity.editor.top_projects",
                 :namefunc="top_editor_projects_namefunc",
                 :hoverfunc="top_editor_projects_hoverfunc",
                 :colorfunc="e => e.data.language",
                 with_limit)
    div(v-if="type == 'top_categories'")
      aw-summary(:fields="$store.state.activity.category.top",
                 :namefunc="e => e.data['$category'].join(' > ')",
                 :colorfunc="e => e.data['$category'].join(' > ')",
                 with_limit)
    div(v-if="type == 'category_tree'")
      aw-categorytree(:events="$store.state.activity.category.top")
    div(v-if="type == 'category_sunburst'")
      aw-sunburst-categories(:data="top_categories_hierarchy", style="height: 20em")
    div(v-if="type == 'timeline_barchart'")
      aw-timeline-barchart(:datasets="datasets", style="height: 100")
    div(v-if="type == 'sunburst_clock'")
      aw-sunburst-clock(:date="date", :afkBucketId="$store.state.activity.buckets.afk[0]", :windowBucketId="$store.state.activity.buckets.window[0]")
</template>

<style lang="scss">
.vis-style-dropdown-btn {
  position: absolute;
  top: 0.8em;
  right: 0.8em;

  .btn {
    border: 0px;
  }
}
</style>

<script>
import _ from 'lodash';
import 'vue-awesome/icons/cog';
import 'vue-awesome/icons/times';
import 'vue-awesome/icons/bars';

import { split_by_hour_into_data } from '~/util/transforms';

// TODO: Move this somewhere else
import { build_category_hierarchy } from '~/util/classes';
import { getColorFromCategory } from '~/util/color';

function pick_subname_as_name(c) {
  c.name = c.subname;
  c.children = c.children.map(pick_subname_as_name);
  return c;
}

export default {
  name: 'aw-selectable-vis',
  props: {
    id: Number,
    type: String,
    editable: { type: Boolean, default: true },
  },
  data: function () {
    return {
      types: [
        'top_apps',
        'top_titles',
        'top_domains',
        'top_urls',
        'top_categories',
        'category_tree',
        'category_sunburst',
        'top_editor_files',
        'top_editor_languages',
        'top_editor_projects',
        'timeline_barchart',
        'sunburst_clock',
      ],
      // TODO: Move this function somewhere else
      top_editor_files_namefunc: e => {
        let f = e.data.file || '';
        f = f.split('/');
        f = f[f.length - 1];
        return f;
      },
      top_editor_files_hoverfunc: e => {
        return 'file: ' + e.data.file + '\n' + 'project: ' + e.data.project;
      },
      // TODO: Move this function somewhere else
      top_editor_projects_namefunc: e => {
        let f = e.data.project || '';
        f = f.split('/');
        f = f[f.length - 1];
        return f;
      },
      top_editor_projects_hoverfunc: e => e.data.project,
    };
  },
  computed: {
    visualizations: function () {
      return {
        top_apps: {
          title: 'Top Applications',
          available:
            this.$store.state.activity.window.available ||
            this.$store.state.activity.android.available,
        },
        top_titles: {
          title: 'Top Window Titles',
          available: this.$store.state.activity.window.available,
        },
        top_domains: {
          title: 'Top Browser Domains',
          available: this.$store.state.activity.browser.available,
        },
        top_urls: {
          title: 'Top Browser URLs',
          available: this.$store.state.activity.browser.available,
        },
        top_editor_files: {
          title: 'Top Editor Files',
          available: this.$store.state.activity.editor.available,
        },
        top_editor_languages: {
          title: 'Top Editor Languages',
          available: this.$store.state.activity.editor.available,
        },
        top_editor_projects: {
          title: 'Top Editor Projects',
          available: this.$store.state.activity.editor.available,
        },
        top_categories: {
          title: 'Top Categories',
          available: this.$store.state.activity.category.available,
        },
        category_tree: {
          title: 'Category Tree',
          available: this.$store.state.activity.category.available,
        },
        category_sunburst: {
          title: 'Category Sunburst',
          available: this.$store.state.activity.category.available,
        },
        timeline_barchart: {
          title: 'Timeline (barchart)',
          available: true,
        },
        sunburst_clock: {
          title: 'Sunburst clock',
          available:
            this.$store.state.activity.window.available &&
            this.$store.state.activity.active.available,
        },
      };
    },
    has_prerequisites() {
      return this.visualizations[this.type].available;
    },
    supports_period: function () {
      if (this.type == 'sunburst_clock' || this.type == 'timeline_barchart') {
        return this.isSingleDay;
      }
      return true;
    },
    top_categories_hierarchy: function () {
      const top_categories = this.$store.state.activity.category.top;
      if (top_categories) {
        const categories = top_categories.map(c => {
          return { name: c.data.$category, size: c.duration };
        });

        return {
          name: 'All',
          children: build_category_hierarchy(categories).map(c => pick_subname_as_name(c)),
        };
      } else {
        return null;
      }
    },
    datasets: function () {
      const METHOD_CATEGORY = 'category';
      const METHOD_ACTIVITY = 'activity';
      const method = METHOD_CATEGORY;
      if (method == METHOD_CATEGORY) {
        const SEP = '>>>';
        const data = this.$store.state.activity.category.by_hour;
        if (data) {
          const categories = new Set(
            Object.values(data)
              .map(result => {
                return result.cat_events.map(e => e.data['$category'].join(SEP));
              })
              .flat()
          );
          const ds = [...categories].map(c_ => {
            const c = this.$store.getters['categories/get_category'](c_.split(SEP));
            if (c) {
              return {
                label: c.name.join(' > '),
                backgroundColor: getColorFromCategory(c, this.$store.state.categories.classes),
                data: Object.values(data).map(results => {
                  const cat = results.cat_events.find(e => _.isEqual(e.data['$category'], c.name));
                  if (cat) return Math.round((cat.duration / (60 * 60)) * 1000) / 1000;
                  else return null;
                }),
              };
            } else {
              console.log('missing c');
            }
          });
          return ds;
        } else {
          return [];
        }
      } else if (method == METHOD_ACTIVITY) {
        const data = split_by_hour_into_data(this.$store.state.activity.active.events);
        return [
          {
            label: 'Total time',
            backgroundColor: '#6699ff',
            data,
          },
        ];
      }
      return [];
    },
    date: function () {
      let date = this.$store.state.activity.query_options.date;
      if (!date) {
        date = this.$store.state.activity.query_options.timeperiod.start;
      }
      return date;
    },
    isSingleDay: function () {
      return _.isEqual(this.$store.state.activity.query_options.timeperiod.length, [1, 'day']);
    },
  },
};
</script>
