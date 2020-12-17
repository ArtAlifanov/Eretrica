<template>
<div style="width: 100%;" v-resize:debounce="keyHandler" :class="{'add-comment-container': newStyle, 'comment-action': action}">
    <div :class="['add-comment', {'with-templates': showTemplates, 'new-style': newStyle},]">
        <media-uploader ref="media" :class="{'show-file-upload':showFileUpload}" :id="0" :audit_id="0" type="requests" layout="grid" v-model="media" :upload-options="uploadOptions" :multiple="false" />
        <el-tooltip v-if="!newStyle" :content="user.name" effect="dark" placement="top-start">
            <ui-avatar :name="user.name" :size="32" :src="user.avatar" />
        </el-tooltip>
        <el-tooltip v-if="type === 'internalNotices' || type === 'request'" :content="$t('general.components.common.media.add_files')" placement="top">
            <svg-icon icon="attachment"  v-if="newStyle" class="action-buttons" @click="handleFileUpload" />
        </el-tooltip>
        <div class="content">
            <el-input 
                autosize 
                ref="content" 
                :class="[{'is-focused': focused, }, newStyle?'new-style':'old-style', 'contentInput']" 
                type="textarea" 
                resize="none" 
                v-model="content" 
                :placeholder="placeholder ? placeholder : $t('general.components.common.add_comment.placeholder')" 
                :disabled="loading || disabled" 
                :validate-event="false"     
                v-resize:debounce="keyHandler"
                @blur="focused = false, handleBlur()" 
                @focus="focused = true" 
                @keydown.native.exact="keyHandler"
                @keydown.native.alt.enter.exact="save"
                @input="(val) => $emit('contentChanged', val)"
            />

            <span class="phantom"></span>

            
            <div 
                v-if="autoSuggest.started" 
                ref="autoSuggestList"
                :key="autoSuggest.yPos" 
                class="auto-suggest-list"
            >
                <div
                    v-if="managerLists.length"
                    :key="item.name + index" 
                    :class="['list-item', {'selected': index === autoSuggest.index}]"
                    @click="handleSelectPM(item)"
                    v-for="(item, index) in managerLists"
                >
                    <ui-avatar :name="item.name" :size="32" :src="item.avatar" />
                    <span class="name" v-html="filterSearch(item.name)"></span>
                    <span class="email" v-html="filterSearch(item.email)"></span>
                </div>
                <div 
                    v-if="autoSuggest.startPos === getEndOfStr(autoSuggest.startPos)"
                    class="list-item"
                >
                    <span class="name no-match">{{ $t('general.input_user_name') }}</span>
                </div>

                <div
                    v-else-if="!managerLists.length"
                    :class="['list-item']"
                >
                    <span class="name no-match">{{ $t('general.no_matches_found') }}</span>
                </div>
            </div>

            <el-dropdown v-if="showTemplates" class="templates" size="small" placement="top-end" trigger="click" @command="onTemplateSelected" @visible-change="onDropdownVisibility">
                <el-tooltip ref="templates-button-tooltip" :content="type == 'internalNotices' ? 'Choose Property maneger and Admin' : $t('general.components.common.add_comment.tooltip_templates')" placement="top-end">
                    <el-button ref="templates-button" class="el-dropdown-link bg-transparent action-buttons" :disabled="loading || disabled">
                        <i class="icon-ellipsis-vert"></i>
                    </el-button>
                </el-tooltip>
                <el-dropdown-menu slot="dropdown">
                    <el-dropdown-item v-for="(template, idx) in templates" :key="template.id" :command="template" :divided="!!idx">
                        {{template.name}}
                        <small style="display: block;color: #A9A9A9;width: 280px;white-space: nowrap;overflow: hidden;text-overflow: ellipsis;">
                            {{template.subject}}
                        </small>
                    </el-dropdown-item>
                    <el-dropdown-item disabled v-if="loadingTemplates">
                        {{$t('general.components.common.add_comment.loading_templates')}}
                    </el-dropdown-item>
                    <el-dropdown-item disabled v-else-if="!loadingTemplates && !templates.length">
                        {{$t('general.components.common.add_comment.empty_templates_placeholder')}}
                    </el-dropdown-item>
                </el-dropdown-menu>
            </el-dropdown>
            <i v-if="newStyle && false" class="el-icon-microphone action-buttons"></i>
            <el-tooltip :content="$t('general.components.common.add_comment.save_shortcut', {shortcut: saveKeysShortcut})" placement="top-end">
                <svg-icon icon="paper_plane" v-if="!loading" class="action-buttons" :disabled="!content || disabled" :loading="loading" @click="save" />
                <i v-else class="el-icon-loading action-buttons"></i>
            </el-tooltip>
        </div>
        <!-- <div v-if="type === 'internalNotices' && content" style="margin: 10px 41px 0 35px;">
            <el-row :gutter="10">
                <el-col :span="24">
                    <el-form-item class="internal-notices-switch">
                        <span>{{ $t('general.components.common.internalnotices.switch_confirm')}}</span>
                        <el-switch v-model="isManagerSelect" @change="resetList"/>
                    </el-form-item>
                </el-col>
                <el-col :span="24">
                    <el-select v-if="isManagerSelect" v-model="selectedManagerLists" multiple filterable remote reserve-keyword :placeholder="$t('general.components.common.internalnotices.input_placeholder')" :remote-method="remoteSearch" :loading="loading" style="width: 100%">
                        <el-option v-for="item in managerLists" :key="item.id" :label="item.name" :value="item.id"></el-option>
                    </el-select>
                </el-col>
            </el-row>
        </div> -->
    </div>
</div>
</template>

<script>
    import {mapActions, mapGetters} from 'vuex'
    import ErrorFallback from 'components/common/AddComment/Error'
    import {displaySuccess, displayError} from 'helpers/messages'
    import { EventBus } from '../../../event-bus.js';
    import resize from 'vue-resize-directive';

    export default {
        directives: {
            resize
        },
        props: {
            id: {
                type: Number,
                required: true
            },
            parentId: {
                type: Number,
            },
            type: {
                type: String,
                required: true,
                validator: type => ['pinboard', 'listing', 'request', 'conversation', 'internalNotices'].includes(type)
            },
            placeholder: {
                type: String
            },
            autofocus: {
                type: Boolean,
                default: false
            },
            reversed: {
                type: Boolean,
                default: false
            },
            showTemplates: {
                type: Boolean,
                default: false
            },
            newStyle: {
                type: Boolean,
                default: false,
            },
            action: {
                type: Boolean,
                default: false
            },
            disabled: {
                type: Boolean,
                default: false,
            }
        },
        data() {
            return {
                content: '',
                focused: false,
                loading: false,
                loadingTemplates: false,
                dropdownTemplatesVisible: false,
                errorFallback: ErrorFallback,
                               
                managerLists: [],
                selectedManagerLists: [],
                loadingList: false,
                isManagerSelect: false,

                autoSuggest: {
                    startPos: 0,
                    started: false,
                    xPos: 0,
                    yPos: 0,
                    visible: false,
                    index: 0,
                    timer: null,
                    blur_timer: null,
                },
                media: [],
                uploadOptions: {
                    drop: true,
                    multiple: true,
                    draggable: true,
                    hideUploadButton: true,
                    extensions: 'vnd.openxmlformats-officedocument.wordprocessingml.document,vnd.openxmlformats-officedocument.spreadsheetml.sheet,pdf,png,jpeg,jpg',
                    hideSelectFilesButton: false,
                    chat: true,
                },
                showFileUpload: false,
            }
        },
        methods: {
            ...mapActions({
                getTemplates: 'getRequestTemplates',
                getPropertyManagers: 'getPropertyManagers',
            }),

            handleFileUpload() {
                this.showFileUpload = true;
                this.$refs.media.selectFiles();
            },
            keyHandler(event) {
                this.$emit('update-dynamic-scroller');
                if(event.key == 'ArrowUp') {
                    if(this.autoSuggest.started)
                        event.preventDefault()
                    if(this.autoSuggest.started && this.autoSuggest.index > 0) {
                        this.autoSuggest.index --;
                        if(this.$refs.autoSuggestList) {
                            let curPos = this.autoSuggest.index * 52;
                            let top = this.$refs.autoSuggestList.scrollTop;
                            let height = this.$refs.autoSuggestList.clientHeight;
                            if(curPos < top )
                                this.$refs.autoSuggestList.scrollBy(0, -52);
                        } 
                    }
                } else if(event.key == 'ArrowDown') {
                    if(this.autoSuggest.started)
                        event.preventDefault()
                    if(this.autoSuggest.started && this.autoSuggest.index < this.managerLists.length - 1) {
                        this.autoSuggest.index ++;

                        if(this.$refs.autoSuggestList) {
                            let curPos = (this.autoSuggest.index + 1) * 52;
                            let top = this.$refs.autoSuggestList.scrollTop;
                            let height = this.$refs.autoSuggestList.clientHeight;
                            if(curPos > top + height)
                                this.$refs.autoSuggestList.scrollBy(0, 52);
                        } 

                    }

                } else if(event.key == 'Enter' || event.key == 'Tab') {
                    if(this.autoSuggest.started) {
                        event.preventDefault();
                        this.selectedManagerLists.push(this.managerLists[this.autoSuggest.index]);
                        this.insertName(this.autoSuggest.startPos, this.managerLists[this.autoSuggest.index].name);
                    }
                } else if(event.key == 'Escape') {
                    if(this.autoSuggest.started) {
                        event.preventDefault();
                    }
                    this.resetAutoSuggest();
                }
            },
            getEndOfStr(startPos) {
                while(this.content.charAt(startPos) !== ' ' && startPos < this.content.length)
                    startPos ++;
                return startPos;
            },
            async remoteSearch(search) {
                this.loadingList = true;
                try {
                    let resp = [];
                    // const respAssignee = await this.getPropertyManagers({request_id: this.$route.params.id});                        
                    // let exclude_ids = [];                                                
                    // respAssignee.data.data.map(item => {
                    //     if(item.type === 'manager'){
                    //         exclude_ids.push(item.edit_id);
                    //     }                                
                    // })
                    // this.selectedManagerLists.forEach((item) => {
                    //     if(this.content.includes('@' + item.name))
                    //         exclude_ids.push(item.id);
                    // });
                    // resp = await this.getPropertyManagers({
                    //     get_all: true,
                    //     search,
                    //     exclude_ids: exclude_ids.join(',')
                    // });
                    resp = await this.axios.get(`alladmins?search=${search}&edit_details=true`);
                    this.managerLists = resp.data.data;
                } catch (err) {
                    displayError(err);
                } finally {
                    this.loadingList = false;
                }   
                this.autoSuggest.visible = true;  
            },
            resetList(){
                this.selectedManagerLists = []
            },
            focus () {
                this.$refs.content.focus()
            },
            onTemplateSelected (template) {
                const caretPosition = this.$refs.content.$el.querySelector('textarea').selectionStart

                this.content = this.content.substring(0, caretPosition) + template.subject + this.content.substring(caretPosition)
                this.$emit('contentChanged', this.content);

                this.focus()
            },
            onDropdownVisibility (state) {
                this.dropdownTemplatesVisible = state

                this.$refs['templates-button-tooltip'].hide()
            },
            async save () {
                if (!/\S/.test(this.content) && !this.media.length) {
                    return
                }

                try {
                    this.loading = true
                    //this.$refs.content.blur()

                    let existingManagerLists = [];
                    let msg = this.content;
                    let media = [...this.media.map((item) => { 
                        return {
                            media:item.file.src, 
                            name:item.file.name
                        }
                    })];

                    this.selectedManagerLists.forEach((item) => {
                        if(this.content.includes('@' + item.name)) {
                            existingManagerLists.push(item.id);
                            if(item.type === 'manager') {
                                msg = msg.replace('@' + item.name, `<a href='#adminPropertyManagersEdit#${item.edit_id}' style="text-decoration:none;color:#8296c5;white-space: nowrap;">@<b>${item.name}</b></a>`);
                            } else if(item.type === 'provider') {
                                msg = msg.replace('@' + item.name, `<a href='#adminServicesEdit#${item.edit_id}' style="text-decoration:none;color:#8296c5;white-space: nowrap;">@<b>${item.name}</b></a>`);
                            }
                        }
                    });
                    
                    let body = this.type !== 'internalNotices' && !this.newStyle ? {
                        id: this.id,
                        comment: this.content,
                        commentable: this.type,
                        parent_id: this.parentId,
                        media: media,
                    } : {
                        id: this.id,
                        parent_id: this.parentId,
                        request_id: this.id,
                        comment: msg,
                        commentable: this.type,
                        media: media,
                        user_ids: existingManagerLists
                    }

                    await this.$store.dispatch('comments/create', body);             

                } catch (error) {
                    displayError(error)
                } finally {
                    this.$emit('addedComment');
                    this.content = ''
                    this.$emit('contentChanged', this.content);
                    this.showFileUpload = false
                    this.media = []
                    this.loading = false
                    this.isManagerSelect = false
                    this.resetList();
                    
                    this.$refs.content.focus()
                }
            },
            insertName(startPos, name) {
                this.content = this.content.substring(0, startPos - 1) + '@' + name + ' ' + this.content.substring(this.getEndOfStr(startPos));
                this.$emit('contentChanged', this.content);
                this.resetAutoSuggest();
            },
            resetAutoSuggest(focus = true) {
                this.managerLists = [];
                this.autoSuggest.index = 0;
                this.autoSuggest.started = false;
                this.autoSuggest.visible = false;
                if(focus)
                    this.focus();
            },
            handleBlur() {
                if(this.autoSuggest.started && !this.autoSuggest.blur_timer)
                    this.autoSuggest.blur_timer = setTimeout(() => {this.resetAutoSuggest(false)}, 100);
            },
            handleSelectPM(pm) {
                if(this.autoSuggest.blur_timer) {
                    clearTimeout(this.autoSuggest.blur_timer);
                    this.autoSuggest.blur_timer = null;
                }
                this.insertName(this.autoSuggest.startPos, pm.name);
                this.selectedManagerLists.push(pm);
            },
            count(str) {
                let matches = str.match(/@/g);
                let count = 0;
                if(matches)
                    count = matches.length;
                return count;
            },
            filterSearch(str) {
                let result = str;
                if(this.autoSuggest.started) {
                    let search = this.content.substring(this.autoSuggest.startPos, this.getEndOfStr(this.autoSuggest.startPos));
                    if(search !== '') {
                        let pos = str.toLowerCase().indexOf(search.toLowerCase());
                        if(pos !== -1) {
                            result = `${str.slice(0, pos)}<b>${str.slice(pos, pos + search.length)}</b>${str.slice(pos + search.length)}`;
                        }
                    }
                }
                return result;
            },
        },
        computed: {
            ...mapGetters({
                user: 'loggedInUser',
                templatesWithId: 'getRequestTemplatesWithId'
            }),

            templates () {
                return this.templatesWithId(this.id) || []
            },
            canShowTemplates () {
                return this.showTemplates && ['request'].includes(this.type)
            },
            saveKeysShortcut () {
                if (navigator.platform.toUpperCase().includes('MAC')) {
                    return 'option+enter'
                }

                return 'alt+enter'
            }
        },
        async mounted () {
            if (this.canShowTemplates) {
                if (!this.templates.length) {
                    try {
                        this.loadingTemplates = true

                        await this.getTemplates({id: this.id})
                    } catch (error) {
                        displayError(error)
                    } finally {
                        this.loadingTemplates = false

                        if (this.dropdownTemplatesVisible) {
                            this.$refs['templates-button'].$el.click()
                        }
                    }
                }
            }

            if (this.autofocus) {
                this.focus()
            }
        },
        watch: {
            content(newStr, oldStr) {
                this.$emit('update-dynamic-scroller');
                if(this.type === 'internalNotices' || this.newStyle) {
                    let elTextArea = this.$refs.content.$el.querySelector('textarea');
                    let curPos = elTextArea.selectionStart;
                    // let inputRect = elTextArea.getBoundingClientRect()
                    // this.autoSuggest.xPos = inputRect.left;
                    // this.autoSuggest.yPos = inputRect.top;
                    // let autoList = this.$el.querySelector('.auto-suggest-list');
                    // autoList.style.left = inputRect.left;
                    // autoList.style.top = inputRect.top ;
                    // console.log(inputRect.left, inputRect.top);
                    if(this.count(newStr) !== this.count(oldStr) && !this.autoSuggest.started && (curPos === 1 || curPos > 1 && !newStr.substr(curPos - 2, 1).match((/[a-zA-Z0-9]@/i)))) {
                        this.autoSuggest.started = true;
                        this.autoSuggest.startPos = curPos;
                    }
                    if(this.count(newStr) < this.count(oldStr))
                        this.resetAutoSuggest();
                    if(this.autoSuggest.started) {
                        if(this.autoSuggest.startPos !== curPos) {
                            if(this.autoSuggest.timer) {
                                clearTimeout(this.autoSuggest.timer);
                                this.autoSuggest.timer = null;
                            }
                            this.autoSuggest.timer = setTimeout(() => {
                                this.remoteSearch(newStr.substring(this.autoSuggest.startPos, this.getEndOfStr(this.autoSuggest.startPos)));
                            }, 100);
                        } else {
                            this.managerLists = [];
                        }
                    }

                    // let curPosition = this.$refs.content.$el.querySelector('textarea').selectionStart;
                    // let phantom = document.querySelector('.phantom');
                    // phantom.innerHTML = this.content.substring(0, curPosition) + '<br>';
                    // this.content.substring(0, curPosition).forEach((val) => console.log(parseInt(val)));
                    // console.log(curPosition);
                }
            },
            "$route": {
                immediate: true,
                handler({path, id}, prevQuery) {
                    if(this.$route.name == "login") {
                        return;
                    }
                    if(this.$route.hash) {
                        let params = this.$route.hash.split('#');
                        this.$router.push({
                            name: params[1],
                            params: {
                                id: parseInt(params[2]),
                            },
                        });
                    }
                }
            },
        }
    }
</script>

<style lang="scss" scoped>
    /deep/ .internal-notices-switch {
        margin-top: 10px;
        .el-form-item__content {
            line-height: 20px !important;
            .el-switch {
                float: right;
            }
        }
    }

    .add-comment-container {
        margin-left: -20px;
        margin-bottom: -15px;
        width: calc(100% + 40px) !important;
    }
    .comment-action {
        position: absolute;
        left: 20px;
        width: 93% !important;
        bottom: 20px;
    }
    .add-comment {
        width: 100%;
        display: flex;
        flex-wrap: wrap;
        align-items: flex-end;
        font-size: 14px;
        position: relative;
        box-sizing: border-box;
        
        &.new-style {
            border-top: 1px solid var(--background-color-base);
            padding: 10px 20px;
        }

        .media {
            width: 100%;
            visibility: hidden;
            position: absolute;

            &.show-file-upload {
                visibility: visible;
                position: relative;
            }
        }

        .action-buttons {
            font-size: 18px;
            width: 24px;
            border: none;
            padding: 0px !important;
            color: #3d3f41;
            box-shadow: none;
            outline: none;

            /deep/ svg {
                font-size: 22px;
            }
            &.el-icon-loading {
                font-size: 24px;
            }
        }

        .content {
            position: relative;
            flex: auto;
            display: flex;
            align-items:flex-end;

            .el-textarea {
                position: relative;
                
                &.old-style {
                    margin-left: 8px;
                    width: calc(100% - 8px);

                    &:before,
                    &:after {
                        content: '';
                        position: absolute;
                        width: 0;
                        height: 0;
                        border-width: 0;
                        border-style: solid;
                        border-color: transparent;
                        border-width: 0 0 8px 6px;
                        transition: border-bottom-color 0.2s cubic-bezier(0.645, 0.045, 0.355, 1);
                    }

                    &:before {
                        left: -6px;
                        bottom: 0;
                    }

                    &:after {
                        left: -4px;
                        bottom: 1px;
                    }

                    &.is-disabled:before {
                        border-bottom-color: #E4E7ED;
                    }

                    &.is-disabled:after {
                        border-bottom-color: #F5F7FB;
                    }

                    &:not(.is-disabled):after {
                        border-bottom-color: #fff;
                    }

                    &:not(.is-disabled).is-focused:before {
                        border-bottom-color: var(--primary-color);
                    }

                    &:not(.is-disabled):not(.is-focused):before {
                        border-bottom-color: #DCDFE6;
                    }

                    &:not(.is-disabled):not(.is-focused):hover:before {
                        border-bottom-color: #C0C4CC;
                    }
                }

                &.new-style {
                    :global(.el-textarea__inner) {
                        background-color: transparent !important;
                        font-family: inherit;
                        font-weight: 400;
                    }
                }

                & + .el-dropdown {
                    position: absolute;
                    bottom: 0;
                    right: -2px;

                    .el-dropdown-link {
                        padding: 9px;
                    }
                }

                &.is-focused :global(.el-textarea__inner)::-webkit-scrollbar-thumb {
                    background-color: #6AC06F;
                    box-shadow: inset -1px -1px 0px darken(#6AC06F, 4%), inset 1px 1px 0px darken(#6AC06F, 4%);
                }

                &:not(.is-focused) :global(.el-textarea__inner) {
                    &:hover::-webkit-scrollbar-thumb {
                        background-color: #C0C4CC;
                        box-shadow: inset -1px -1px 0px darken(#C0C4CC, 4%), inset 1px 1px 0px darken(#C0C4CC, 4%);
                    }

                    &:not(:hover)::-webkit-scrollbar-thumb {
                        background-color: #DCDFE6;
                        box-shadow: inset -1px -1px 0px darken(#DCDFE6, 4%), inset 1px 1px 0px darken(#DCDFE6, 4%);
                    }
                }

                :global(.el-textarea__inner) {
                    padding: 6px 8px;
                    border-radius: 12px;
                    max-height: 256px;
                    overflow-y: overlay;
                    overflow-x: hidden;
                    scrollbar-width: thin;
                    overscroll-behavior: contain;
                    border-bottom-left-radius: 0;
                    -webkit-appearance: none;
                    -webkit-overflow-scrolling: touch;

                    &::placeholder {
                        color: #3d3f41;
                        font: 'Radikal';
                    }

                    &::-webkit-scrollbar {
                        width: 14px;
                    }

                    &::-webkit-scrollbar-thumb {
                        border: 4px transparent solid;
                        background-clip: padding-box;
                        border-radius: 12px;
                    }

                    &::-webkit-scrollbar-thumb:window-inactive {
                        background-color: lighten(#6AC06F, 16%);
                    }
                }
            }

            & + .el-button {
                margin-left: 16px;
            }

            .auto-suggest-list {
                position: absolute;
                bottom: 40px;
                left: 10px;
                z-index: 999;
                width: 450px;
                max-height: 300px;
                overflow: auto;
                background-color: var(--color-white);
                border: 1px solid var(--border-color-base);
                border-radius: 8px 8px 5px 5px;
                .list-item {
                    padding: 10px 30px 10px 10px;
                    cursor: pointer;

                    &.selected {
                        background-color: var(--border-color-lighter);
                    }
                    &:hover {
                        background-color: var(--border-color-lighter);
                    }
                    .name {
                        margin-left: 15px;
                        color: var(--color-black);
                        text-transform: capitalize;
                        font-size: 15px;

                        &.no-match {
                            color: var(--color-text-placeholder);
                        }
                    }
                    .email {
                        margin-left: 15px;
                        color: var(--color-text-primary);
                        font-size: 12px;
                    }
                }
            }
        }

        &.with-templates {
            .content {
                .el-textarea {
                    :global(.el-textarea__inner) {
                        padding-right: 32px;
                    }
                }
            }
        }

        .phantom {
            position: absolute;
            left: 0;
            top: 0;
            width: 100%;
            height: 100%;
            z-index: -999;
            padding: 6px 8px;
        }

    }

</style>
