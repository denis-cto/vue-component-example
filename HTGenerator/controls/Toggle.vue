<template>
    <q-toggle
            :label="data.label"
            checked-icon="check"
            color="primary"
            dense
            unchecked-icon="clear"
            v-model="model"
    ></q-toggle>
</template>

<script>
    export default {
        name: 'hg-ui-toggle',
        props: {
            data: {
                type: Object,
            },
        },
        data: function () {
            return {
                model: this.data.value || false
            }
        },
        watch: {
            model(newValue) {
                if (this.data.callback) {
                    this.data.callback(newValue);
                } else {
                    this.$emit('input', newValue);
                }
            }
        },
        created() {
            console.log('TOGGLE PROPS', this.data);
        }
    }
</script>
