<template>
    <q-input
            :label="data.label"
            clear-icon="close"
            clearable
            dense
            min="0"
            outlined
            step="1"
            type="number" v-model.number="model"
    ></q-input>
</template>

<script>
    import _ from 'lodash';

    export default {
        name: 'hg-ui-input-number',
        props: {
            data: {
                type: Object,
            },
        },
        data: function () {
            return {
                model: this.data.value || null
            }
        },
        watch: {
            model(value) {
                if (this.data.callback) {
                    this.data.callback(value);
                } else {
                    const option = (_.isNil(value) || value === '') ? null : {
                        collection: this.data.collection,
                        generateHashtag: this.data.generateHashtag,
                        label: `Index [${value}]`,
                        value,
                    };
                    this.$emit('input', option);
                }
            }
        }
    }
</script>
