<template>
    <div class="ui upload-drop file-manager" v-loading="loading">
        <aside :class="`ui__sidebar ${isSidebarOpen ? 'ui__sidebar_open' : 'ui__sidebar_closed'}`">
            <div @click="handleSidebar" class="aside-controller">
                <i :class="`sidebar-toggle ${isSidebarOpen ? 'el-icon-caret-left' : 'el-icon-caret-right'}`" />
            </div>

            <div class="ui__sidebar-inner">
                <el-tree
                        class="folder-tree"
                        :data="foldersTree"
                        :key="treeKey"
                        node-key="id"
                        :default-expanded-keys="defaultExpandedKeys"
                        :expand-on-click-node="false">
                    <template slot-scope="{ node, data }">
                        <span :class="`folder ${data.id == foldersStates.is_openId ? 'folder_open' : ''}`" >
                            <span class="folder__icon">
                                <svg-icon icon="folder" />
                            </span>
                            <el-input v-if="data.id == foldersStates.is_renameId"
                                      :ref="'renameFolder_'+data.id"
                                      v-model="data.name"
                                      @blur="updateFolder(data)"></el-input>
                            <span v-else
                                  @click="openFolder(data.id)"
                                  class="folder__name">
                                {{ data.name }}
<!--                                - id:{{data.id}} - {{node.parent.data.id}}-->
                            </span>

                            <span v-if="data.can_edit && data.can_create_folder && data.can_delete"
                                  :class="`folder__menu ${activeFolderMenu === data.id ? 'active' : ''}`">
                                <el-dropdown size="small" trigger="click" placement="bottom-end" @visible-change="handleFolderMenuActive($event, data.id)">
                                    <el-tooltip :content="$t('general.actions.label')">
                                        <el-button class="folder-tree__dropdown-link">
                                            <i class="icon-ellipsis-vert"></i>
                                        </el-button>
                                    </el-tooltip>
                                    <el-dropdown-menu slot="dropdown">
                                        <el-dropdown-item v-if="data.can_edit" @click.native="renameFolder(data.id)">
                                            <span class="folder__menu-icon">
                                                <svg-icon icon="add" />
                                            </span>
                                            {{$t('models.file_manager.rename_folder')}}
                                        </el-dropdown-item>
                                        <el-dropdown-item v-if="data.can_create_folder" @click.native="addFolder({can_delete: 1, name: 'folder', parent_id: +data.id})">
                                            <span class="folder__menu-icon">
                                                <svg-icon icon="edit" />
                                            </span>
                                            {{$t('models.file_manager.add_folder')}}
                                        </el-dropdown-item>
                                        <el-dropdown-item v-if="data.can_delete" @click.native="delFolder(data)">
                                            <span class="folder__menu-icon">
                                                <svg-icon icon="trash" />
                                            </span>
                                            {{$t('models.file_manager.delete_folder')}}
                                        </el-dropdown-item>
                                    </el-dropdown-menu>
                                </el-dropdown>
                            </span>
                        </span>
                    </template>
                </el-tree>
            </div>
        </aside>

        <main class="ui__main">
            <div class="ui__menu">
                <div class="file-path">
                    <div class="file-path__nav">
                        <button class="nav-btn prev"
                                :disabled="openedFolderPath.length === 0"
                                @click="goUpFolder()"><i class="icon-left"></i></button>
                        <button class="nav-btn next"
                                :disabled="!openedFolder.children"
                                @click="goDownFolder()"><i class="icon-right"></i></button>
                    </div>


                    <div class="file-path__row">
                        <div class="file-path__item"
                             @click="openFolder(0)">
                            <svg-icon icon="home" />
                        </div>
                        <template v-if="openedFolderPath.length"
                             v-for="(folder, index) in openedFolderPath">
                            <div :class="`file-path__item ${(openedFolderPath.length - index) > 3 ? 'hidden' : ''}`"
                                 :key="folder.id"
                                 @click="openFolder(folder.id)">
                                <span v-if="openedFolderPath.length - index == 3">...</span>
                                <span v-else>{{folder.name}}</span>
                            </div>
                        </template>
                    </div>

                    <div class="file-path__btns">
                        <el-tooltip
                                :content="$t('models.file_manager.add_to_root')"
                                class="item" effect="light"
                                placement="bottom"
                        >
                            <div class="add-rootfolder" @click="addFolder({can_delete: 1, name: 'to root', model_id: model_id})">
                                <svg-icon icon="folder" />
                                <div class="plus">
                                    <svg-icon icon="add" />
                                </div>
                            </div>
                        </el-tooltip>
                        <el-tooltip
                                :content="$t('models.file_manager.upload_file')"
                                class="item" effect="light"
                                placement="bottom"
                        >
                            <div class="upload-file" @click="handleShowMediaDialog"><i class="el-icon-upload"/></div>
                        </el-tooltip>
                    </div>
                </div>
            </div>

            <div :class="`ui__content custom-scroller ${isInDragAreaCounter > 0 && openedFolder.can_upload ? 'ui__content_dragover' : ''}`"
                 @dragenter="dragenter"
                 @dragleave="dragleave">
                <div class="upload-wrapper"
                     v-show="isInDragAreaCounter > 0 && openedFolder.can_upload"
                     @dragenter="dragenter"
                     @dragleave="dragleave">
                    <upload-document @fileUploaded="uploadFile" class="drag-custom" drag multiple
                                     accept-type=".pdf, .doc, .docx, .xls, .xlsx, .png, .jpg, .jpeg" fullWidth
                    />
                </div>
                <div v-if="openedFolder.children" class="folders">
                    <el-row>
                        <el-col :span="12" v-if="openedFolder" v-for="folder in openedFolder.children" :key="folder.id">
                            <div class="folder-item" @click="openFolder(folder.id)">
                            <span class="folder-item__icon">
                                <svg-icon icon="folder" />
                            </span>
                                <span class="folder-item__name">{{folder.name}}</span>
                            </div>
                        </el-col>
                    </el-row>
                </div>
                <div v-if="openedFolder.files" class="files">
                    <el-row>
                        <el-col :span="12" v-if="openedFolder" v-for="file in openedFolder.files" :key="file.id">
                            <div class="file-item">
                                <a :href="file.url" target="_blank" :title="file.file_name" class="file-item__link">
                                <span class="file-item__icon">
                                    <i :class="returnFileIcon(file.file_name.split('.')[1])"></i>
                                </span>

                                    <span class="file-item__name">{{file.name}}</span>
                                </a>

                                <div class="file-item__del" @click="deleteFolderFile(file)">
                                    <el-tooltip
                                            :content="$t('general.actions.delete')"
                                            class="item" effect="light"
                                            placement="top"
                                    >
                                        <svg-icon icon="trash" />
                                    </el-tooltip>
                                </div>
                            </div>
                        </el-col>
                    </el-row>
                </div>
            </div>
        </main>
    </div>
</template>

<script>
import {displayError, displaySuccess} from "helpers/messages";
import {mapActions, mapGetters} from 'vuex';
import UploadDocument from 'components/UploadDocument';

export default {
    name: 'file-manager',
    components: {
        UploadDocument,
    },
    props: {
        showMediaDialog: Function,
        model_id: Number,
        getFoldersList: {
            type: Function,
            default: null
        }
    },
    data(){
        return{
            loading: true,
            isSidebarOpen: false,
            activeFolderMenu: '',
            foldersData: [],
            treeKey: 0,
            openedFolderPath: [],
            openedFolderHistory: [{id: 0, active: true}],
            foldersStates: {
                is_openId: '',
                is_renameId: '',
            },
            isInDragArea: false,
            isInDragAreaCounter: 0,
            filesCounter: 0,
        }
    },
    methods: {
        ...mapActions([
            'getResidentFileManager',
            'getResidentFolders',
            'getResidentFolder',
            'addResidentFolder',
            'updateResidentFolder',
            'deleteResidentFolder',
            'deleteMediaFile',
            'uploadMediaFile',
        ]),
        dragenter () {
            this.isInDragAreaCounter += 1;
            this.isInDragArea = true;
        },
        dragleave () {
            this.isInDragAreaCounter -= 1;
            this.isInDragArea = false;
        },
        uploadFile(file) {
            this.filesCounter += 1;

            let media = {
                folder_id: this.openedFolder.id,
                media: file.src,
                name: file.name
            };

            this.uploadMediaFile({id: this.model_id, media}).then(resp => {
                this.filesCounter -= 1;
                if(this.filesCounter === 0) {
                    this.updateOpenedFolderFilesIfChanged(resp.data.folder_id);
                }
            });

            this.isInDragAreaCounter = 0;
            this.isInDragArea = false;
        },
        returnFileIcon (format) {
            switch (format) {
                case 'pdf':
                    return 'icon-file-pdf';
                case 'doc':
                case 'docx':
                case 'xls':
                case 'xlsx':
                    return 'icon-file-excel';
                default:
                    return 'icon-file-image';
            }
        },
        handleSidebar(){
            this.isSidebarOpen = !this.isSidebarOpen
        },
        handleFolderMenuActive (e, id) {
            this.activeFolderMenu = e ? id : '';
        },
        goUpFolder() {
            let length = this.openedFolderPath.length;

            if(length <= 1) {
                this.foldersStates.is_openId = '';
            } else {
                this.foldersStates.is_openId = this.openedFolderPath[length - 1 - 1].id;
            }
        },
        goDownFolder() {
            this.openFolder(this.openedFolder.children[0].id);
        },
        insertArrInFoldersData (arr) {
            if(arr.length > 0) {
                arr.forEach(item => {
                    let isFind = (_.find(this.foldersData, (folder) => {return folder.id == item.id}));

                    if ( !isFind ) {
                        this.foldersData.push(item);
                    } else {
                        let $obj = this.foldersData.find(folder => folder.id == item.id);
                        $obj = {...item, children: $obj.children};
                    }
                });

                let addedIds = _.map(arr, 'id');
                let parentId = arr[0].parent_id;
                let byParentIds = _.filter(this.foldersData, item => {return item.parent_id == parentId});
                    byParentIds = _.map(byParentIds, 'id');
                let rmIds = _.difference(byParentIds, addedIds);

                rmIds.forEach(rmId => {
                    _.remove(this.foldersData, item => item.id == rmId);
                });

                _.sortBy(this.foldersData, ['level', 'id']);
            }
        },
        addFolder(query) {
            this.addResidentFolder(query);

            this.openFolder(query.parent_id, true);
        },
        updateFolder(data) {
            this.foldersStates.is_renameId = '';
            this.openFolder(data.id);

            this.updateResidentFolder({folder_id: data.id, ...data});

            this.treeKey += 1;
        },
        delFolder(data) {
            this.deleteResidentFolder({folder_id: data.id}).then(resp => {
                this.openFolder(data.parent_id);

                this.treeKey += 1;
            });
        },
        openFolder(id) {
            this.foldersStates.is_openId = id;

            if (id !== 0) {
                this.getResidentFolder({folder_id: id, include_files: true}).then(item => {
                    let getedFolders = item.folders;
                    getedFolders.forEach(folder => folder.parent_id == null ? folder.parent_id = id : '');
                    this.insertArrInFoldersData(getedFolders);

                    let opened = this.foldersData.find(folder => folder.id == this.foldersStates.is_openId);
                    opened.files = item.files;

                    this.treeKey += 1;
                })
            }
        },
        renameFolder(id) {
            this.foldersStates.is_renameId = id;

            this.treeKey += 1;

            this.$nextTick(() => {
                this.$refs['renameFolder_'+id].focus();
            });
        },
        async handleShowMediaDialog () {
            let foldersTreeCopy = [];
            let flatTree = [];

            await this.getResidentFileManager({resident_id : this.model_id, include_qbu_folders: true}).then(item => {
                foldersTreeCopy = item.folders;
            });

            function processTree(tree, d = 1) {
                let newTreeItem = tree;

                if (typeof newTreeItem.folders !== 'undefined' && newTreeItem.folders.length > 0) {
                    flatTree.push(newTreeItem);
                    newTreeItem.folders.forEach((item) => {
                        processTree(item, d++);
                    });
                } else {
                    flatTree.push(newTreeItem);
                }
            }

            foldersTreeCopy.forEach((item) => {
                processTree(item, 1);
            });

            this.getFoldersList(flatTree);
            this.showMediaDialog();
        },
        updateOpenedFolderFilesIfChanged(folder_id) {
            let opened = this.foldersData.find(folder => folder.id == this.foldersStates.is_openId);

            if(opened.id === folder_id) {
                this.getResidentFolder({folder_id: folder_id, include_files: true}).then(item => {
                    opened.files = item.files;

                    this.treeKey += 1;
                });
            }
        },
        async deleteFolderFile(file) {
            try {
                let res = await this.deleteMediaFile({
                    id: this.model_id,
                    media_id: file.id
                });
                if(res.success) {
                    displaySuccess(res);
                    this.updateOpenedFolderFilesIfChanged(file.folder_id);
                }
            } catch( err ) {
                displayError(err);
            }
        },
    },
    computed: {
        defaultExpandedKeys() {
            return [this.foldersStates.is_openId, this.foldersStates.is_renameId];
        },
        foldersTree() {
            let tree = [];
            let folders = this.foldersData,
                obj = {};

            folders.sort((a, b) => {
                return (a.parent_id || -1) - (b.parent_id || -1) || a.level - b.level;
            });

            tree = folders.filter(a => {
                a.children = obj[a.id] && obj[a.id].children;
                obj[a.id] = a;

                if (a.parent_id === null) {
                    return true;
                }

                obj[a.parent_id] = obj[a.parent_id] || {};
                obj[a.parent_id].children = obj[a.parent_id].children || [];
                obj[a.parent_id].children.push(a);
            });
            this.treeKey += 1;

            return tree;
        },
        openedFolder() {
            this.openedFolderPath = [];
            let obj = this.foldersData.find(item => item.id == this.foldersStates.is_openId);

            if(!!obj) {
                let path = [];
                let level = obj.level;

                path.push(obj);
                for(let i = 0; i < level - 1; i++) {
                    let parent = this.foldersData.find(item => item.id == path[i].parent_id);
                    path.push(parent);
                }
                path.reverse();

                this.openedFolderPath = path;
            }

            return !!obj ? obj : { children: this.foldersData.filter(item => item.level == 1)};
        },
    },
    created() {
        this.loading = true;

        this.getResidentFolders({resident_id: this.model_id}).then(item => {
            this.insertArrInFoldersData(item.folders);
            this.foldersDataTree = item.folders;

            this.loading = false;
        });
    },
}

</script>

<style lang="scss" scoped>
$primary: #47cedf;
$secondary: #c298cf;
$black: #787878;
$border-radius: 5px;

$bp: 900px;

.file-manager {
    box-sizing: border-box;
    *,
    *::before,
    *::after {
        box-sizing: inherit;
    }
}

a {
    text-decoration: none;
    color: inherit;
}

[draggable] {
  user-select: none;
  -khtml-user-drag: element;
  -webkit-user-drag: element;
}



/*.folder-tree::-webkit-scrollbar {*/
/*    display: none;*/
/*}*/



.ui {
    position: relative;
    display: flex;
    margin: 0 auto;
    max-width: $bp;
    background: #fff;
    overflow: hidden;
    
    @media (min-width: $bp) {
        // margin: 10vh auto 0;
        min-height: 200px;
        height: 100%;
        box-shadow: 0 0 20px rgba($black, 0.1);
        border-radius: $border-radius;
    }
    
    &.upload-drop--dragover {
        .ui__sidebar {
            opacity: .5;
        }

        .ui__main {
            transition: border .5s ease;
            border: 4px dashed $primary;

            > * {
                pointer-events: none;
            }
        }
    }
}
.aside-controller{
    cursor:pointer;
    transition: all .5s ease;
}
.sidebar-toggle{
    border: 2px solid;
    font-size: 11px;
    padding: 0 1px;
    border-radius: 3px;
}
.ui__sidebar {
    position: relative;
    display: block;
    background: #f1f0f1;
    color: var(--color-text-regular);
    transition: all .5s ease;
    max-width: 300px;
    max-height: 100%;
    &-inner {
        overflow: auto;
        padding: 15px 5px;
        height: 100%;
        transition: all .5s ease;
    }

    .aside-controller {
        z-index: 1;
        position: absolute;
        top: 10px;
        right: 12px;
        &:hover {
            color: var(--primary-color);
        }
    }
    &_closed &-inner {
        overflow: hidden;
        padding: 0;
    }
    &_closed {
        width: 0;
        padding: 0;
        .aside-controller {
            right: -30px;
        }
        + .ui__main .file-path {
            padding-left: 40px;
        }
    }
    &_open {
        width: 20%;
        @media (min-width: $bp) {
            display: block;
            width: 180px;
        }
    }
}

.ui__main {
    flex: 1 1 auto;
    display: flex;
    flex-direction: column;
    width: calc(100% - 300px);
}

.ui__menu {
    display: flex;
}



.file-path {
    flex: 1 1 auto;
    display: flex;
    align-items: center;
    margin: 0;
    padding: 8px 16px;
    font-size: .875rem;
    background: #f8f8f8;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
    border-radius: 0 0 3px 3px;

    &__nav {
        margin-right: 10px;
        display: flex;
        align-items: center;
        .nav-btn {
            cursor: pointer;
            border: none;
            outline: none;
            background: transparent;
            font-size: 16px;
            &:disabled {
                cursor: default;
            }
            &:hover:not(:disabled) {
                color: var(--primary-color);
            }
        }
        .next {
            margin-left: 8px;
        }
    }
    &__row {
        display: flex;
        justify-content: flex-end;
    }
    &__btns {
        margin-left: auto;
        display: flex;
        align-items: center;
        .add-rootfolder,
        .upload-file {
            cursor: pointer;
            &:hover {
                color: var(--primary-color);
            }
        }
        .add-rootfolder {
            position: relative;
            font-size: 15px;
            .plus {
                position: absolute;
                left: 0;
                right: 0;
                top: 0;
                bottom: 0;
                display: flex;
                align-items: center;
                justify-content: center;
                padding-top: 1.35em;
                font-size: 0.35em;
            }
        }
        .upload-file{
            font-size: 19px;
            margin: 0 0 0 14px;
        }
    }
    &__item {
        cursor: pointer;
        &:not(:last-child) {
            margin-right: 4px;
        }
        &:after {
            content: '/';
            display: inline-block;
            margin-left: 4px;
        }
        &:hover {
            color: var(--primary-color);
        }
        &.hidden {
            display: none;
            &:last-of-type {
                display: block;
                content: '...';
            }
        }
    }
}


.folder {
    display: flex;
    align-items: center;
    color: var(--color-text-regular);
    &__icon {
        margin-right: 6px;
        display: inline-block;
    }
    &__menu:not(.active) {
        opacity: 0;
    }
    &__menu-icon {
        margin-right: 4px;
        font-size: 12px;
    }
    &_open {
        .folder__name {
            color: var(--color-primary);
        }
    }
    .el-icon-folder{
        color: #47cedf;
        margin-right:5px;
    }
    .el-icon-delete{
        color: #fff;
        margin-right:5px;
    }
    &:hover &__menu {
        opacity: 1;
    }
}

.ui__content {
    overflow: auto;
    padding: 20px 15px;
    flex: 1;
    &_dragover {
        position: relative;
        &:after {
            content: '';
            position: absolute;
            z-index: 1;
            left: 0;
            right: 0;
            top: 0;
            bottom: 0;
            width: 100%;
            height: 100%;
        }
        .upload-wrapper {
            z-index: 99;
        }
    }
}

.upload-wrapper {
    z-index: -1;
    position: absolute;
    top: 0;
    left: 15px;
    width: calc(100% - 30px);
    background: rgba(#fff, 0.7);
    /deep/ .drag-custom {
        .el-upload {
            .el-upload-dragger {
                width: calc(400% + 25px);
                height: 180px;
                color: var(--color-primary);
                background-color: var(--primary-color-lighter);
                border-color: var(--color-primary);
            }
        }
    }
}


.files {
    position: relative;
    margin-top: 25px;
}
.folder-item,
.file-item {
    cursor: pointer;
    display: inline-flex;
    align-items: center;
    padding: 4px 8px;
    max-width: 100%;
    background: transparent;
    transition: all 0.3s;
    &:hover {
        /*background: var(--color-main-background-base);*/
        color: var(--color-primary);
    }
    &__icon {
        font-size: 2em;
        line-height: 1;
    }
    &__name {
        padding-top: 4px;
        margin-left: 6px;
    }
}
.file-item {
    &__del {
        padding: 4px 4px 0;
        opacity: 0;
    }
    &:hover &__del {
        opacity: 1;
    }

    &__link {
        display: flex;
        align-items: center;
        max-width: 90%;
        flex: 1;
        min-width: 0;

        .file-item__name {
            white-space: nowrap;
            overflow: hidden;
            text-overflow: ellipsis;
        }
    }
}
</style>
<style lang="scss">
    .folder-tree {
        max-width: 100%;
        background-color: transparent;
        .el-tree-node {
            &.is-current {
                > .el-tree-node__content {
                    background-color: transparent;
                    /*background-color: var(--color-primary-lighter);*/
                }
            }
        }
        .el-tree-node__content {
            &:hover {
                background: rgba(255, 255, 255, 0.3);
            }
        }
        .el-tree-node>.el-tree-node__children {
            overflow: visible;
            .folder__name {
                display: inline-block;
                overflow: hidden;
            }
        }
        .el-dropdown .el-button {
            margin: 0 10px;
            padding: 0 !important;
        }
        &__dropdown-link.el-button--default {
            margin-left: 10px;
            background-color: transparent;
            color: var(--color-text-placeholder);
            &:focus,
            &:hover {
                box-shadow: none;
                background-color: transparent;
                color: var(--color-primary);
            }
        }
    }
</style>
