<script>
/* global Flash */

import eventHub from '../event_hub';
import { getParameterByName } from '../../lib/utils/common_utils';
import loadingIcon from '../../vue_shared/components/loading_icon.vue';
import { COMMON_STR } from '../constants';

import groupsComponent from './groups.vue';

export default {
  components: {
    loadingIcon,
    groupsComponent,
  },
  props: {
    store: {
      type: Object,
      required: true,
    },
    service: {
      type: Object,
      required: true,
    },
    hideProjects: {
      type: Boolean,
      required: true,
    },
  },
  data() {
    return {
      isLoading: true,
      isSearchEmpty: false,
      searchEmptyMessage: '',
    };
  },
  computed: {
    groups() {
      return this.store.getGroups();
    },
    pageInfo() {
      return this.store.getPaginationInfo();
    },
  },
  methods: {
    fetchGroups({ parentId, page, filterGroupsBy, sortBy, archived, updatePagination }) {
      return this.service.getGroups(parentId, page, filterGroupsBy, sortBy, archived)
                .then((res) => {
                  if (updatePagination) {
                    this.updatePagination(res.headers);
                  }

                  return res;
                })
                .then(res => res.json())
                .catch(() => {
                  this.isLoading = false;
                  $.scrollTo(0);

                  Flash(COMMON_STR.FAILURE);
                });
    },
    fetchAllGroups() {
      const page = getParameterByName('page') || null;
      const sortBy = getParameterByName('sort') || null;
      const archived = getParameterByName('archived') || null;
      const filterGroupsBy = getParameterByName('filter') || null;

      this.isLoading = true;
      // eslint-disable-next-line promise/catch-or-return
      this.fetchGroups({
        page,
        filterGroupsBy,
        sortBy,
        archived,
        updatePagination: true,
      }).then((res) => {
        this.isLoading = false;
        this.updateGroups(res, Boolean(filterGroupsBy));
      });
    },
    fetchPage(page, filterGroupsBy, sortBy, archived) {
      this.isLoading = true;

      // eslint-disable-next-line promise/catch-or-return
      this.fetchGroups({
        page,
        filterGroupsBy,
        sortBy,
        archived,
        updatePagination: true,
      }).then((res) => {
        this.isLoading = false;
        $.scrollTo(0);

        const currentPath = gl.utils.mergeUrlParams({ page }, window.location.href);
        window.history.replaceState({
          page: currentPath,
        }, document.title, currentPath);

        this.updateGroups(res);
      });
    },
    toggleChildren(group) {
      const parentGroup = group;
      if (!parentGroup.isOpen) {
        if (parentGroup.children.length === 0) {
          parentGroup.isChildrenLoading = true;
          // eslint-disable-next-line promise/catch-or-return
          this.fetchGroups({
            parentId: parentGroup.id,
          }).then((res) => {
            this.store.setGroupChildren(parentGroup, res);
          }).catch(() => {
            parentGroup.isChildrenLoading = false;
          });
        } else {
          parentGroup.isOpen = true;
        }
      } else {
        parentGroup.isOpen = false;
      }
    },
    leaveGroup(group, parentGroup) {
      const targetGroup = group;
      targetGroup.isBeingRemoved = true;
      this.service.leaveGroup(targetGroup.leavePath)
        .then(res => res.json())
        .then((res) => {
          $.scrollTo(0);
          this.store.removeGroup(targetGroup, parentGroup);
          Flash(res.notice, 'notice');
        })
        .catch((err) => {
          let message = COMMON_STR.FAILURE;
          if (err.status === 403) {
            message = COMMON_STR.LEAVE_FORBIDDEN;
          }
          Flash(message);
          targetGroup.isBeingRemoved = false;
        });
    },
    updatePagination(headers) {
      this.store.setPaginationInfo(headers);
    },
    updateGroups(groups, fromSearch) {
      this.isSearchEmpty = groups ? groups.length === 0 : false;
      if (fromSearch) {
        this.store.setSearchedGroups(groups);
      } else {
        this.store.setGroups(groups);
      }
    },
  },
  created() {
    this.searchEmptyMessage = this.hideProjects ?
      COMMON_STR.GROUP_SEARCH_EMPTY : COMMON_STR.GROUP_PROJECT_SEARCH_EMPTY;

    eventHub.$on('fetchPage', this.fetchPage);
    eventHub.$on('toggleChildren', this.toggleChildren);
    eventHub.$on('leaveGroup', this.leaveGroup);
    eventHub.$on('updatePagination', this.updatePagination);
    eventHub.$on('updateGroups', this.updateGroups);
  },
  mounted() {
    this.fetchAllGroups();
  },
  beforeDestroy() {
    eventHub.$off('fetchPage', this.fetchPage);
    eventHub.$off('toggleChildren', this.toggleChildren);
    eventHub.$off('leaveGroup', this.leaveGroup);
    eventHub.$off('updatePagination', this.updatePagination);
    eventHub.$off('updateGroups', this.updateGroups);
  },
};
</script>

<template>
  <div>
    <loading-icon
      class="loading-animation prepend-top-20"
      size="2"
      v-if="isLoading"
      :label="s__('GroupsTree|Loading groups')"
    />
    <groups-component
      v-if="!isLoading"
      :groups="groups"
      :search-empty="isSearchEmpty"
      :search-empty-message="searchEmptyMessage"
      :page-info="pageInfo"
    />
  </div>
</template>
