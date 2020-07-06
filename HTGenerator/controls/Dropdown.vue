<template>
    <q-select
            :label="data.label" :options="options" @filter="onFilter" dense
            fill-input hide-selected input-debounce="100"
            options-dense outlined use-input
            v-model="model"
    >
        <template v-slot:append>
            <q-icon
                    @click.stop="model = null"
                    class="cursor-pointer"
                    name="clear"
                    v-if="model !== null"
            />
        </template>
        <template v-slot:no-option>
            <q-item>
                <q-item-section class="text-grey">
                    No results
                </q-item-section>
            </q-item>
        </template>
    </q-select>
</template>

<script>
    export default {
        name: 'hg-ui-dropdown',
        props: {
            data: {
                type: Object,
            },
        },
        data: function () {
            return {
                model: this.data.value,
                options: [],
            };
        },
        watch: {
            'data.value': {
                handler(newValue) {
                    if (!newValue) this.model = newValue;
                }
            },
            model(newValue) {
                this.$emit('input', newValue);
            },
        },
        methods: {
            onFilter(value, update) {
                update(() => {
                    const needle = value.toLowerCase();
                    this.options = this.data.collection
                        .filter(item => item.label.toLowerCase().indexOf(needle) > -1);
                });
            }
        }
    }
</script>
