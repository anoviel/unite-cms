<template>
    <div class="uk-flex-1" style="min-width: 180px; max-width: 360px;">
        <form class="uk-search uk-search-default uk-search-rounded uk-width-expand" v-if="filterRules.length > 0" @submit.prevent="applySearch(searchInput)">
            <button type="submit" class="uk-icon uk-search-icon">
                <icon v-if="!advancedFilter" name="search" />
                <icon v-else name="x-circle" class="uk-text-danger" />
            </button>
            <input id="view-filter-search" class="uk-search-input" type="search" :placeholder="$t(advancedFilter ? 'content.list.search.placeholder_filter' : 'content.list.search.placeholder')" v-model="searchInput">
            <button :title="$t('content.list.filter.title')" uk-tooltip type="button" class="uk-search-icon-flip uk-icon uk-search-icon" :class="{ 'uk-text-primary': hasFilterRules }" @click="tmpFilter = Object.assign(value)"><icon name="command" /></button>
        </form>
        <modal v-if="tmpFilter" @hide="tmpFilter = null" :title="$t('content.list.filter.title')">
            <form slot class="uk-padding-small" @submit.prevent="applyFilter(tmpFilter)">
                <filter-rule :rules="filterRules" v-model="tmpFilter" />
            </form>
            <div slot="footer" uk-grid>
                <div class="uk-text-left uk-flex-1">
                    <button class="uk-button uk-button-danger" type="button" @click="applyFilter()">{{ $t('content.list.filter.clear') }}</button>
                </div>
                <div class="uk-text-right uk-flex-1">
                    <button class="uk-button uk-button-default uk-modal-close" type="button">{{ $t('content.list.filter.cancel') }}</button>
                    <button class="uk-button uk-button-primary" type="button" @click="applyFilter(tmpFilter)">{{ $t('content.list.filter.apply') }}</button>
                </div>
            </div>
        </modal>
    </div>
</template>
<script>

    import Icon from '../../Icon';
    import Modal from "../../Modal";
    import FilterRule from "./_filterRule";

    const normalizeFilter = function(filter){

        if(!filter) {
            return null;
        }

        if(filter.field && filter.operator) {
            return {
                field: filter.field,
                operator: filter.operator,
                path: filter.path || undefined,
                cast: filter.cast || undefined,
                value: filter.value,
            }
        } else if(filter.AND && filter.AND.length > 0) {
            let AND = filter.AND.map(normalizeFilter).filter(f => f !== null);
            return AND.length > 0 ? { AND } : null;
        } else if(filter.OR && filter.OR.length > 0) {
            let OR = filter.OR.map(normalizeFilter).filter(f => f !== null);
            return OR.length > 0 ? { OR } : null;
        } else {
            return null;
        }
    };

    export default {
        components: {FilterRule, Icon, Modal},
        data() {
            return {
                searchInput: null,
                advancedFilter: false,
                tmpFilter: null
            }
        },
        props: {
            value: Object,
            view: Object,
        },
        watch: {
            value(val) {
                if(!this.hasFilterRules) {
                    this.advancedFilter = false;
                    this.searchInput = null;
                }
            }
        },
        computed: {
            hasFilterRules() {
                return Object.keys(this.value).length > 0;
            },
            filterRules() {

                if(!this.view) {
                    return [];
                }

                let rules = [];
                this.view.fields.forEach((field) => {
                    let component = this.$unite.getListFieldType(field);
                    if(component.filter) {
                        rules = [...rules, ...component.filter(field, this.view, this.$unite)];
                    }
                });
                return rules;
            },
            filterLabels() {
                return {
                    matchType: this.$t('content.list.search.filters.matchType'),
                    matchTypes: [
                        {"id": "AND", "label": this.$t('content.list.search.filters.matchTypes.AND')},
                        {"id": "OR", "label": this.$t('content.list.search.filters.matchTypes.OR')}
                    ],
                    "addRule": this.$t('content.list.search.filters.addRule'),
                    "removeRule": this.$t('content.list.search.filters.removeRule'),
                    "addGroup": this.$t('content.list.search.filters.addGroup'),
                    "removeGroup": this.$t('content.list.search.filters.removeGroup'),
                    "textInputPlaceholder": this.$t('content.list.search.filters.textInputPlaceholder'),
                };
            }
        },
        methods: {
            applySearch(value) {
                this.advancedFilter = false;
                this.$emit('input', !value ? {} : {
                    OR: this.filterRules.filter((rule) => { return rule.searchable }).map((rule) => {
                        return {
                            field: rule.id,
                            path: rule.path || undefined,
                            operator: 'CONTAINS',
                            value: value
                        };
                    }),
                });
            },
            applyFilter(filter = null) {
                this.searchInput = null;
                this.advancedFilter = true;
                this.$emit('input', Object.assign({}, normalizeFilter(filter)));
                this.tmpFilter = null;
            }
        }
    }
</script>
