<template>
    <div class="filters" :class="{[layout]: true}">
        <template v-for="field in fields">
            <!-- <label v-if="field.title">{{(field.title)}}</label> -->
            <!-- <component :is="field.type" v-bind="field.props" v-on="field.listeners" v-model.sync="data[field.name]">                
                <component v-if="((field.children) && (field.type === 'el-select'))" v-for="child in field.children" :key="child.name" :is="child.type" :label="(child.props.label)" :value="child.props.value" />
                <component v-else-if="field.children" v-for="child in field.children" :key="child.name" :is="child.type" v-bind="child.props"/>
            </component> -->
            <div :class="['item', {'active': child.props.value === data['event']}]" v-for="(child, index) in field.children" :key="child.name" @click="handleClick(child)">
               {{ $t(child.props.label) }}
            </div>
        </template>
    </div>
</template>

<script>
    export default {
        props: {
            schema: {
                type: Array,
                required: true
            },
            data: {},
            layout: {
                type: String,
                default: 'column',
                validator: layout => ['row', 'column'].includes(layout)
            }
        },
        methods: {
            reset () {
                this.$emit('update:data', Object.keys(this.data).reduce((acc, curr) => {
                    acc[curr] = null

                    return acc
                }, {}))
            },
            handleClick(item) {
                this.data['event'] = item.props.value;
            }
        },
        computed: {
            fields () {
                return this.schema.reduce((fields, currField) => {
                    if (currField.require && !this.data[currField.require]) {
                        return fields;
                    }

                    fields.push(currField);

                    return fields;
                }, []);
            }
        },
        watch: {
            data: {
                deep: true,
                handler (filters) {
                    this.$emit('changed', Object.entries(filters).reduce((acc, [key, value]) => {
                        if (value) {
                            acc[key] = value;
                        }
                        return acc;
                    }, {}));
                }
            }
        }
    };
</script>

<style lang="scss">
    .filters {
        max-height: 150px;
        overflow: auto;
        color: lighten(#000, 32%);

        &.row {
            display: flex;
            flex-wrap: wrap;
            justify-content: space-between; 
            > * {
                width: calc(100% * (1/2));
                &:not(:nth-last-child(-n+2)) {
                    margin-bottom: 8px;
                }
            }
            > div {
                justify-content: flex-end;
            }
        }
        &.column {
            display: flex;
            flex-direction: column;
            label {
                margin: 4px 0;
            }
        }
        &::-webkit-scrollbar{
            width: 8px;
        }
        &::-webkit-scrollbar-thumb{
            background-color: #e2e2e2;
            border: 1px solid transparent;
            border-radius: 11px;
            background-clip: content-box;
        }
        &::-webkit-scrollbar * {
            background: transparent;
        }

        .item {
            padding: 5px 0px 5px 15px;
            cursor: pointer;
            
            &:hover {
                background: #f8f8f8;
            }
            &.active {
                background: #f0f0f0;
            }
        }
    }
</style>
