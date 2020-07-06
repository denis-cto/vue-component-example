<template>
    <q-card class="hashtag-generator" style="width: 480px;">
        <q-card-section>
            <div class="row items-center">
                <div class="text-h6">Hashtag Generator</div>
                <q-space></q-space>
                <q-btn dense flat icon="close" round v-close-popup></q-btn>
            </div>
        </q-card-section>

        <q-separator></q-separator>

        <q-card-section class="scroll" style="min-height: 340px; max-height: 50vh">
            <q-select
                    :hide-dropdown-icon="!!currentApplication" :options="options_applications"
                    :readonly="!!currentApplication"
                    @filter="(event, update) => onFilter(event, update, 'applications')" dense fill-input
                    input-debounce="100" label="Application"
                    options-dense outlined
                    use-input v-model="modelApplication"
            >
                <template v-slot:selected-item="scope">
                    <span class="text-grey">(ID: {{ scope.opt.value }})</span>
                </template>
                <template v-slot:option="scope">
                    <q-item
                            v-bind="scope.itemProps"
                            v-on="scope.itemEvents"
                    >
                        <q-item-section>
                            <q-item-label class="row">
                                {{ scope.opt.label }}
                                <q-space></q-space>
                                <span class="text-grey">(ID: {{ scope.opt.value }})</span>
                            </q-item-label>
                        </q-item-section>
                    </q-item>
                </template>
                <template v-slot:no-option>
                    <q-item>
                        <q-item-section class="text-grey">
                            No results
                        </q-item-section>
                    </q-item>
                </template>
            </q-select>

            <q-separator class="q-mt-md q-mb-md"></q-separator>

            <component
                    :data="data"
                    :is="controls[data.controlName]" :key="$ind"
                    @input="onChangeHashtag($event, $ind)" class="q-mt-md" v-for="(data, $ind) in stackOfDropdowns"
            ></component>
        </q-card-section>

        <div class="q-pa-md hashtag-generator__preview">
            <div class="text-caption text-uppercase q-mb-sm">Preview</div>
            <q-select
                    :options="options_objects" @filter="(event, update) => onFilter(event, update, 'objects')" dense
                    fill-input hide-selected input-debounce="100" label="Select Object"
                    options-dense outlined
                    use-input v-model="modelObject"
            >
                <template v-slot:no-option>
                    <q-item>
                        <q-item-section class="text-grey">
                            No results
                        </q-item-section>
                    </q-item>
                </template>
            </q-select>
            <pre class="q-mt-md" v-if="previewHashtag">{{ previewHashtag }}</pre>
        </div>

        <q-card-section>
            <q-input
                    dense label="Resulting hashtag" outlined readonly
                    ref="resultingHashtag" v-model="resultingHashtag.tag"
            >
                <q-popup-proxy anchor="center middle" no-focus self="center middle" v-model="isCopied">
                    <q-banner class="bg-grey-7 text-white" dense rounded>
                        {{ copiedMsg }}
                    </q-banner>
                </q-popup-proxy>
                <template v-slot:after>
                    <q-btn
                            :disabled="!resultingHashtag.tag" @click="onCopy" color="primary"
                            flat
                            icon="file_copy"
                    >
                        <q-tooltip anchor="top middle" self="bottom middle">
                            Copy to the Clipboard
                        </q-tooltip>
                    </q-btn>
                </template>
            </q-input>
        </q-card-section>

        <q-separator/>

        <q-card-section align="right">
            <q-btn flat label="Cancel" no-caps v-close-popup/>
            <q-btn
                    :disabled="!resultingHashtag.tag || !lastFocusEl" @click="onInsert" color="positive"
                    label="Insert at cursor" no-caps
                    unelevated v-close-popup
            ></q-btn>
        </q-card-section>
    </q-card>
</template>

<script>
    import MakeHttp from 'general/utils/MakeHttp';
    import _ from 'lodash';
    import insertTextAtCursor from 'insert-text-at-cursor';
    import controls from './controls';

    const http = MakeHttp();

    export default {
        name: 'hashtag-generator',
        data: function () {
            const {id: value, name: label} = (MOBSTEDAPP.application || {});
            const lastFocusEl = window.lastFocusEl;
            return {
                lastFocusEl,
                isCopied: false,
                copiedMsg: '',
                applications: [], // all applications received from the server
                currentApplication: MOBSTEDAPP.application || null,
                modelApplication: (value && label) ? {label, value} : null, // value for select
                options_applications: (value && label) ? [{label, value}] : [], // options for select

                hashtagsInfo: null,
                stackOfDropdowns: null,
                nameGroupHashtag: null,
                resultingHashtag: {path: null, tag: null},

                objects: [],
                modelObject: null,
                options_objects: [],
                previewHashtag: null,

                scope: null,
                filterName: null,

                apiProviderMethodsResponse: null,
                typeResponse: null,
            };
        },
        computed: {
            controls: () => controls,
        },
        watch: {
            'modelApplication.value': {
                async handler(newValue) {
                    if (newValue) {
                        await Promise.all([
                            this.getHashtags(),
                            this.getDataByKey({key: 'object', alias: 'objects'}),
                        ]);
                    }
                }
            },
            'modelObject.value': 'changeHashtagValue',
            'resultingHashtag.tag': 'changeHashtagValue',
            isCopied(value) {
                if (value) setTimeout(() => this.isCopied = false, 3000);
            }
        },
        async created() {
            await Promise.all([
                this.getDataByKey({key: 'applications'}),
                this.getDataByKey({key: 'object', alias: 'objects'}),
                this.getHashtags(),
            ]);
        },
        methods: {
            async onCopy() {
                const text = this.resultingHashtag.tag;
                const defaultMessage = 'Hashtag was copied to the clipboard';
                this.selectResultingHashtag();

                // Fallback
                if (!navigator.clipboard) {
                    try {
                        const successful = document.execCommand('copy');
                        if (!successful) throw new Error();
                        this.copiedMsg = defaultMessage;
                        this.isCopied = true;
                    } catch (err) {
                        this.copiedMsg = 'Could not copy text';
                        this.isCopied = true;
                    }
                    return;
                }

                try {
                    await navigator.clipboard.writeText(text);
                    this.copiedMsg = defaultMessage;
                    this.isCopied = true;
                } catch (err) {
                    this.copiedMsg = 'Could not copy text';
                    this.isCopied = true;
                }
            },
            selectResultingHashtag() {
                const el = this.$refs.resultingHashtag;
                el.focus();
                el.select();
            },
            onInsert() {
                insertTextAtCursor(this.lastFocusEl, this.resultingHashtag.tag);
            },

            // TODO: will split to two methods (get Objects and Applications data)
            async getDataByKey({key, alias = null}) {
                const applicationId = (this.modelApplication || {}).value;
                if (applicationId && key == 'applications' || !applicationId && key == 'object') return Promise.resolve();

                const pageSize = 100;
                const rest = (key == 'object') ? {applicationId, pageSize} : {pageSize};
                const storeName = alias || key;

                try {
                    // get the first page
                    const res = await http.get(`/${key}`, {page: 1, ...rest});
                    if (!res.data || !res.meta) throw new Error();

                    const {meta} = res;
                    let data = res.data;

                    // get all pages
                    if (meta && meta.current_page !== meta.last_page) {
                        let requests = [];
                        for (let page = 2; page <= meta.last_page; page++) {
                            requests.push(http.get(`/${key}`, {page, ...rest}));
                        }
                        const resRequests = await Promise.all(requests);
                        data = _.flatMap([...[res], ...resRequests], item => item.data);
                    }

                    this[storeName] = data.map(
                        item => ({
                            label: key == 'object' ? `ID: ${item.id}` : item.attributes.Name,
                            value: item.id
                        })
                    );
                } catch (e) {
                    this.$q.notify({
                        message: `[${key}] data could not be retrieved`,
                        color: 'negative'
                    });
                }
            },

            onFilter(value, update, context) {
                update(() => {
                    const needle = value.toLowerCase();
                    this[`options_${context}`] = this[context]
                        .filter(item => item.label.toLowerCase().indexOf(needle) > -1);
                });
            },

            async getHashtags() {
                const applicationId = (this.modelApplication || {}).value;
                if (!applicationId) return;

                try {
                    const res = await http.get('/hashtags/info', {applicationId});
                    if (!res.data) throw new Error();
                    let collection = res.data.map(
                        item => ({
                            label: item.type.replace('TagsInfo', ''),
                            value: item.type.replace('TagsInfo', ''),
                            collection: item.attributes.availableTags,
                        })
                    );

                    const index = _.findIndex(collection, item => item.value === 'Event');
                    if (index != -1) {
                        let lastEventOption = _.cloneDeep(collection[index]);
                        collection[index].label = 'Events (Available in Triggers\' Operations only)';
                        lastEventOption.label = 'LastEvent (Available everywhere except Triggers\' Operations)';
                        lastEventOption.value = 'LastEvent';
                        collection.push(lastEventOption);
                    }

                    collection = _.sortBy(collection, ['value']);

                    this.hashtagsInfo = _.reduce(collection, (acc, item) => {
                        acc[item.value] = item.collection;
                        return acc;
                    }, {});

                    this.stackOfDropdowns = [{
                        controlName: 'hg-ui-dropdown',
                        label: 'Category',
                        value: null,
                        collection
                    }];
                } catch (e) {
                    console.error(e);
                    this.$q.notify({
                        message: 'Hashtag data could not be retrieved',
                        color: 'negative'
                    });
                }
            },

            async onChangeHashtag(selectedOption, index) {
                this.resultingHashtag = {path: null, tag: null};
                const lastIndex = this.stackOfDropdowns.length - 1;

                // If reset dropdown value
                if (!selectedOption) {
                    // Deletion of dropdowns other than the current one and all up to the current one
                    this.stackOfDropdowns.splice(index + 1, lastIndex - index);
                    // Reset value of current dropdown
                    this.stackOfDropdowns[index].value = null;
                    this.generateHashtag();
                    return;
                }

                const {
                    controlName: selectedControlName,
                    isFilterName,
                    generateHashtag,
                    label: selectedLabel,
                    value: selectedValue,
                } = selectedOption;
                const label = `Property of ${selectedLabel}`; // TODO: translate it!

                if (index === 0) this.nameGroupHashtag = selectedValue;
                this.stackOfDropdowns[index].value = selectedValue;

                const scopesFilter = ['ObjectsFilter', 'EventsFilter', 'ListsFilter'];
                if (_.includes(scopesFilter, selectedValue))
                    this.scope = selectedValue.replace('Filter', '').toLowerCase();

                if (isFilterName) this.filterName = selectedValue;

                const collection = await this.prepareCollection(selectedOption);

                if (lastIndex !== index) this.stackOfDropdowns.splice(index + 1, lastIndex - index);

                // Hashtag is already
                if (!collection || generateHashtag) {
                    this.generateHashtag();
                    if (!generateHashtag) return;
                }

                const controlName = selectedControlName || 'hg-ui-dropdown';

                // New level of dropdowns
                this.stackOfDropdowns.push({controlName, label, value: null, collection, generateHashtag});
            },

            generateHashtag() {
                const {nameGroupHashtag} = this;
                let minLvl = 1;
                if (nameGroupHashtag == 'Filter' || nameGroupHashtag == 'Event') {
                    minLvl = 2;
                }
                if (nameGroupHashtag == 'Operation') {
                    minLvl = 3;
                }

                let pathArray = this.stackOfDropdowns
                    .filter(({value}) => value !== 'Filter' && !_.isNil(value))
                    .map(({value}) => value);

                if (nameGroupHashtag == 'Operation') {
                    let index = _.findIndex(pathArray, item => item === 'Response');
                    if (index != -1 && _.includes(pathArray, 'Result')) {
                        pathArray.splice(index + 2, 0, 0);
                    }
                    index = _.findIndex(pathArray, item => item === 'skip');
                    if (index != -1) {
                        pathArray.splice(index, 1);
                    }
                }

                if (pathArray.length <= minLvl) return;

                if (nameGroupHashtag == 'Event' || nameGroupHashtag == 'LastEvent') {
                    if (pathArray[1] == 'columns') {
                        pathArray.splice(1, 1);
                    }
                }

                const path = pathArray.join('.');
                const tag = `#${pathArray.join(':')}#`;
                this.resultingHashtag = {path, tag};
            },

            async prepareCollection(selectedOption) {
                const {
                    collection: options,
                    value: selectedValue,
                    isFilter,
                    isResponse,
                    isResult,
                } = selectedOption;

                if (isResponse) {
                    this.typeResponse = selectedValue;
                }

                if (isResult) {
                    const responces = _.filter(this.apiProviderMethodsResponse, response => response.responseType == this.typeResponse);
                    return _.map(responces, response => ({
                        label: response.responseName,
                        value: 'skip',
                        generateHashtag: true,
                        collection: this.prepareResponse(response.response),
                    }));
                }

                //#Operation:NewOperation:Response:Result:0:newResponse:address:geo:lat#

                if (selectedOption.APIProviderMethodId) {
                    const apiMethodId = selectedOption.APIProviderMethodId;
                    await this.getApiProviderMethodsResponse(apiMethodId);
                }

                const modifierFilter = ['Data', 'Value', 'Count', 'PrecentChange'];
                const selfModifierFilter = ['SelfData', 'SelfValue', 'SelfCount', 'SelfPrecentChange'];

                if (selectedValue === 'Operation') {
                    const typeResponseCollection = [
                        {label: 'Response', value: 'Response'},
                        {label: 'Error', value: 'Error'},
                    ];
                    const innerResponse = [
                        {label: 'Code', value: 'Code'},
                        {label: 'Result', value: 'Result'},
                    ];
                    const operations = await this.getOperations();
                    return operations.map(operation => {
                        return {
                            ...operation,
                            collection: typeResponseCollection.map(type => {
                                return {
                                    ...type,
                                    isResponse: true,
                                    collection: innerResponse.map(resp => {
                                        const isResult = (resp.value === 'Result');
                                        return {
                                            ...resp,
                                            isResult
                                        }
                                    })
                                }
                            })
                        }
                    });
                }
                if (isFilter) {
                    if (selectedValue === 'Data') {
                        if (this.scope == 'lists') return null;
                        const columns = await this.getFilterColumns();
                        return columns.map(item => ({label: item, value: item}));
                    }

                    if (_.includes(selfModifierFilter, selectedValue)) {
                        let collection = null;
                        if (selectedValue === 'SelfData') {
                            if (this.scope !== 'lists')
                                collection = await this.getFilterColumns();
                        }
                        return this.hashtagsInfo.Object.map(item => ({
                            label: item,
                            value: item,
                            controlName: 'hg-ui-input-number',
                            generateHashtag: !!collection,
                            collection
                        }));
                    }
                }
                if (selectedValue === 'ListsFilter') {
                    const filterModifierCollection = [...modifierFilter, ...selfModifierFilter].map(item => {
                        return {
                            label: item,
                            value: item,
                            generateHashtag: false,
                            collection: null,
                            isFilter: (item == 'Data' || item == 'SelfData')
                        }
                    });
                    return options.map(item => ({
                        ...item,
                        collection: filterModifierCollection,
                    }));
                }
                if (selectedValue === 'Filter') {
                    const filterModifierCollection = [...modifierFilter, ...selfModifierFilter].map(item => {
                        return {
                            label: item,
                            value: item,
                            controlName: item == 'Data' ? 'hg-ui-input-number' : null,
                            generateHashtag: (item == 'Data'),
                            isFilter: (_.includes([...selfModifierFilter, 'Data'], item))
                        }
                    });
                    const collection = options.map(item => ({
                        label: item,
                        value: item,
                        isFilterName: true,
                        collection: filterModifierCollection,
                    }));
                    return [
                        {label: 'Objects', value: 'ObjectsFilter', collection},
                        {label: 'Events', value: 'EventsFilter', collection},
                        {label: 'Lists', value: 'ListsFilter', collection},
                    ];
                }
                if (selectedValue === 'Backendname') {
                    const collection = ['label', 'value', 'selected'];
                    return options.map(item => {
                        let rest = {};
                        if (item.count) {
                            rest.collection = [];
                            for (let i = 0; i < item.count; i++) {
                                rest.collection.push({
                                    label: `Index [${i}]`,
                                    value: String(i),
                                    collection
                                });
                            }
                        }
                        return {
                            label: item.backendName,
                            value: item.backendName,
                            ...rest,
                        }
                    });
                }
                if (_.isPlainObject(options)) {
                    return Object.keys(options)
                        .map(item => ({
                            label: item,
                            value: item,
                            collection: options[item]
                        }));
                }
                if (_.isArray(options)) {
                    return options.map(item => {
                        if (_.isPlainObject(item)) {
                            return {
                                label: item.label,
                                value: item.value,
                                collection: item.collection || null,
                                controlName: item.controlName || null,
                                isFilter: item.isFilter || null,
                                isResult: item.isResult || null,
                                isResponse: item.isResponse || null,
                                isFilterName: item.isFilterName || null,
                                generateHashtag: item.generateHashtag || false,
                            }
                        } else {
                            return {label: item, value: item}
                        }
                    });
                }
                return false;
            },

            prepareResponse(response) {
                try {
                    const parsed = JSON.parse(response);

                    const generateOption = property => {
                        const res = Object.keys(property).map(key => {
                            const collection = _.isObject(property[key]) ? generateOption(property[key]) : null;
                            return {
                                label: key,
                                value: key,
                                generateHashtag: !!collection,
                                collection,
                            }
                        });

                        return res;
                    };

                    return generateOption(parsed);
                } catch (e) {
                    console.log(e);
                    return response;
                }
            },

            async getApiProviderMethodsResponse(apiMethodId) {
                try {
                    const {data} = await http.get('/apiprovidermethods/response', {apiMethodId});
                    const apiProviderMethodsResponse = ((data || {}).attributes || {}).Responces;
                    if (!apiProviderMethodsResponse) throw new Error();

                    this.apiProviderMethodsResponse = apiProviderMethodsResponse;

                    return Promise.resolve(apiProviderMethodsResponse);
                } catch (e) {
                    console.log(e);
                    return Promise.resolve([]);
                }
            },

            async getOperations() {
                try {
                    const {data} = await http.get('/operations/list');
                    if (!data || data.length < 1) throw new Error();

                    const columns = data.map(item => ({
                        label: item.attributes.Name,
                        value: item.attributes.Name,
                        APIProviderMethodId: item.attributes.APIProviderMethodId,
                    }));

                    return Promise.resolve(columns);
                } catch (e) {
                    console.log(e);
                    return Promise.resolve([]);
                }
            },

            async getFilterColumns() {
                try {
                    const {filterName, scope} = this;
                    const applicationId = (this.modelApplication || {}).value;
                    if (!filterName || !applicationId || !scope) throw new Error();

                    const res = await http.post('/filters/info/columns', {filterName, scope, applicationId});
                    const {columns} = (res.data || {}).attributes || {};
                    if (!columns) throw new Error();

                    return Promise.resolve(columns);
                } catch (e) {
                    console.log(e);
                }
            },

            async getValueHashtagFromAPI() {
                const nameGroupHashtag = this.nameGroupHashtag == 'Filter' ?
                    `${_.upperFirst(this.scope)}Filter` : this.nameGroupHashtag;
                const applicationId = Number((this.modelApplication || {}).value);
                const objectId = Number((this.modelObject || {}).value);
                const {path, tag} = this.resultingHashtag;
                const extraParams = {
                    // ..._.omit(rootState.hashtags, ['ObjectsFilter', 'EventsFilter', 'ListsFilter', 'Operation', 'List']),
                    Globals: {applicationId, objectId}
                };
                const options = {
                    Application: {
                        url: '/hashtags/applications',
                        data: {
                            ids: [applicationId],
                        }
                    },
                    Object: {
                        url: '/hashtags/objects',
                        data: {ids: [objectId], applicationId},
                    },
                    ObjectsFilter: {
                        url: '/hashtags/filters/objects',
                        data: {applicationId, objectId, extraParams},
                    },
                    EventsFilter: {
                        url: '/hashtags/filters/events',
                        data: {applicationId, objectId, extraParams},
                    },
                    ListsFilter: {
                        url: '/hashtags/filters/lists',
                        data: {applicationId, objectId, extraParams},
                    },
                    List: {
                        url: '/hashtags/list',
                        data: {applicationId},
                    },
                    System: {
                        url: '/hashtags/system',
                        data: {applicationId},
                    },
                    Tax: {
                        url: '/hashtags/taxes',
                        data: {applicationId},
                    },
                    Tenant: {
                        url: '/hashtags/tenant',
                        data: {}
                    },
                };
                const includesGroup = ['ObjectsFilter', 'EventsFilter', 'ListsFilter'];
                let tags = [];
                let tagData = tag;

                if (_.includes(includesGroup, nameGroupHashtag)) {
                    // TODO: Data (pagination)
                    const tag = `#${_.slice(path.split('.'), 0, 3).join(':')}#`;
                    tagData = {tag, objectId}; //pagination: { page: 1, pageSize: 10 }
                }
                if (nameGroupHashtag == 'List') {
                    // TODO: Data (pagination)
                    tagData = {tag}; //pagination: { page: 1, pageSize: 10 }
                }

                tags.push(tagData);

                try {
                    const props = options[nameGroupHashtag];
                    if (!props) throw new Error();
                    const res = await http.post(props.url, {...props.data, tags});
                    let apiHashtags = {};

                    if (res && !_.isEmpty(res.data)) {
                        const resData = _.isArray(res.data) ? res.data : [res.data];
                        _.each(resData, v => {
                            let keyApiHashtags = v.type;
                            let data = v.attributes;

                            if (v.attributes.tag && v.attributes.tag.search('Filter') != -1) {
                                const paramsHashTag = _.trim(v.attributes.tag, '#').split(':');
                                let value = _.lowerFirst(paramsHashTag[2]);
                                const path = _.join(_.drop(paramsHashTag), '.');
                                keyApiHashtags = paramsHashTag[0];
                                if (value.search(/selfData|selfValue|selfCount|selfPrecentChange/) != -1) {
                                    value = _.lowerFirst(value.replace('self', ''));
                                }
                                data = _.set({}, path, Object.freeze(data[value]));
                            }

                            if (keyApiHashtags && keyApiHashtags.toLowerCase() == 'list') {
                                const paramsHashTag = _.trim(v.attributes.tag, '#').split(':');
                                const path = _.join(_.drop(paramsHashTag), '.');
                                keyApiHashtags = paramsHashTag[0];
                                data = _.set({}, path, Object.freeze(data.data));
                            }

                            apiHashtags[keyApiHashtags] = _.merge(apiHashtags[keyApiHashtags], data);
                        });
                    }

                    this.previewHashtag = _.get(apiHashtags, path, '');
                } catch (e) {
                    console.info(e);
                }
            },

            changeHashtagValue(value) {
                if (!value) {
                    this.previewHashtag = null;
                } else {
                    if (this.resultingHashtag.tag) {
                        this.selectResultingHashtag();
                        this.getValueHashtagFromAPI();
                    }
                }
            }
        }
    }
</script>

<style lang="stylus">
    @import '~stylus/quasar/variables.styl'

    .hashtag-generator__preview
    background-color $ light
</style>
