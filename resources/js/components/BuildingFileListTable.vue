<template>
    <div>
        <no-data-placeholder v-if="!items.length" />
        <el-table
            v-else
            :key="JSON.stringify(items)"
            :data="items"
            style="width: 100%"
            :show-header="false"
            >
            <!-- <div slot="empty">
                <el-alert                                     
                    :title="$t('general.no_data_available')"
                    type="info"
                    show-icon
                    :closable="false"
                    class="no_data_box"
                >
                </el-alert>
            </div> -->
            <el-table-column
                width="40"
                align="right"
            >
                <template slot-scope="scope">
                    <svg-icon :icon="getExtention(scope.row.file_name)" />
                </template>
            </el-table-column>

            <el-table-column
                align="left"
            >
                <template slot-scope="scope" >
                    {{scope.row.name}}
                </template>
            </el-table-column>

            <el-table-column
                prop="category"
                width="180"
            >
                <template slot-scope="scope">
                    <strong>{{$t(`models.building.media_category.${scope.row.category}`)}}</strong>
                </template>
            </el-table-column>
            
            <el-table-column
                width="170"
                align="right"
            >
                <template slot-scope="scope">
                    <el-tooltip
                        :content="$t('general.actions.update')"
                        :class="['item', {'hide': !editMode}]" effect="light" placement="top-end"
                    >
                        <el-button style="background-color: transparent;  color: rgb(132, 132, 132);   box-shadow: none !important;    padding-left: 13px;" type="danger" round @click="editFiles(scope.$index)" size="mini">
                            <svg-icon icon="edit" />
                        </el-button>
                    </el-tooltip>
                    <el-tooltip
                        :content="$t('general.actions.edit')"
                        :class="['item', {'hide': !editMode}]" effect="light" placement="top-end"
                    >
                        <el-button style="background-color: transparent;  color: rgb(132, 132, 132);   box-shadow: none !important;    padding-left: 13px;" type="danger" round @click="handleView(scope.row.url)" size="mini">
                            <svg-icon icon="search" />
                        </el-button>
                    </el-tooltip>
                    <el-tooltip
                        :content="$t('general.actions.delete')"
                        :class="['item', {'hide': !editMode}]" effect="light" placement="top-end"
                    >
                        <el-button style="background-color: transparent;  color: rgb(132, 132, 132);   box-shadow: none !important;    padding-left: 13px;" type="danger" round @click="deleteFile(scope.$index)" size="mini">
                            <svg-icon icon="trash" />
                        </el-button>
                    </el-tooltip>
                </template>
            </el-table-column>
        </el-table>
    </div>
</template>

<script>

    import uuid from 'uuid/v1'
    import {mapActions, mapGetters} from 'vuex';

    import {ResponsiveMixin} from 'vue-responsive-components'

    import NoDataPlaceholder from 'components/NoDataPlaceholder'

    export default {
        name: 'BuildingFileListTable',
        mixins: [
            ResponsiveMixin
        ],
        components: {
            NoDataPlaceholder,
        },
        props: {
            items: {
                type: Array,
                default: () => {
                    return [];
                }
            },
            editMode: {
                type: Boolean,
                default: () => {
                    return false;
                }
            },
            type: {
                type: String,
                default: () => {
                    return ''
                }
            },
            editFiles: {
                type: Function,
                required: true,
            }
        },
        data () {
            return {
                clicks: [],
                delay: 700,
                icons: {
                    'pdf': 'pdf',
                    'doc': 'word', 
                    'docx': 'word', 
                    'xls': 'excel', 
                    'xlsx': 'excel', 
                }
            }
        },
        methods: {
            deleteFile( index ) {
                this.clicks[index] ++
                if(this.clicks[index] === 1) {
                    this.$emit('delete-document', 'media', index)
                    this.timer = setTimeout(() => {
                        this.clicks[index] = 0
                    }, this.delay);
                }
            },
            handleView(url) {
                window.open(url, '_blank');
            },
            getExtention(name) {
                let icon = 'folder'
                if(name) {
                    let ext = name.split('.').pop().toLowerCase();
                    let isSupport = ['pdf', 'doc', 'docx', 'xls', 'xlsx'].includes(ext);
                    if(isSupport)
                        icon = this.icons[ext];
                }

                return icon;
            }
        },
        computed: {
            ...mapGetters('application', {
                constants: 'constants'
            }),
        },
        mounted() {
            this.clicks = this.items.map(item => 0)
        },
        watch: {
            'items': {
                immediate: false,
                handler () {
                    // TODO - auto blur container if visible is true first
                    this.clicks = this.items.map(item => 0);
                }
            },
        },
    }
</script>

<style lang="scss" scoped>

    /deep/ .file-name {
        color: var(--primary-color);
        text-decoration: none;
    }

    /deep/ .file-name:hover {
        text-decoration: none;
        color: var(--primary-color-lighter);
    }

    /deep/ .el-table {
        &::before {
            background-color: transparent;
        }
        tr:hover td {
            background-color: transparent;
        }
        td {
            padding: 8px 0px 0px;
            border: none;
            font-weight: 700;
            color: var(--text-color);

            .cell {
                padding: 15px 10px;
                background-color: #f8f8f8;

                svg {
                    font-size: 20px;
                }

                .el-button {
                    padding: 0 5px;
                    svg {
                        font-size: 20px;
                        color: var(--text-color);
                    }
                }
                .hide {
                    visibility: hidden;
                }
            }

            &:first-of-type .cell {
                border-radius: 12px 0 0 12px
            }
            &:last-of-type .cell {
                border-radius: 0 12px 12px 0;
            }
        }
    }
    
</style>
