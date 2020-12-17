<template>
  <div class="media-gallery" v-if="media.length">
    <el-row>
      <el-col :span="8" v-for="(file, index) in media" :key="index">
        <div :class="['document-item', {'active': index === activeDoc && selectedFile === activeDoc}]" @click="selectDocument(file, index)">
          <div class="document-icon">
            <svg-icon icon="pdf" />
          </div>
          <div class="document-title">
            {{$t(file.name)}}
          </div>
        </div>
      </el-col>
    </el-row>
  </div>
  <placeholder :src="require('img/5c98a90bb5c05.png')" v-else-if="!media.length">
    {{$t('resident.no_data.media')}}
    <small>{{$t('resident.media_info')}}</small>
  </placeholder>
</template>

<script>
  import Placeholder from './Placeholder'
  import isotope from 'vueisotope'
  import {mapActions, mapGetters} from 'vuex'

  export default {
    props: {
        media: {
            type: Array,
            default: []
        },
        selectedFile: {
            type: Number,
            default: -1
        },
        limit: {
            type: Number,
            default: 0
        }
    },
    components: {
      Placeholder,
      isotope
    },
    data () {
      return {
          option: {
              getSortData: {
                  id: "id"
              },
              sortBy : "id"
          },
          activeDoc: '',
          doc_files: []
      }
    },
    methods: {
        canFileBePreviewed (file_name) {
          if (file_name) {
              const ext = file_name.split('.').pop();
              return ext;
          } else {
              return ''
          }
        },
        selectDocument(file, index) {
          const type = this.canFileBePreviewed(file.file_name);
          if (type === 'pdf') {
            this.activeDoc = index;
            this.$emit('onSelectDoc', file, index);
          } else {
            window.open(file.url, '_blank');
          }
        },

    },
    computed: {
        ...mapGetters('application', {
            constants: 'constants'
        }),
    }
  }

</script>

<style>
    .media-item-fade-enter-active,
    .media-item-fade-leave-active {
        transition: opacity .5s;
    }
    .media-item-fade-enter,
    .media-item-fade-leave-to {
        opacity: 0;
    }
</style>

<style lang="scss" scoped>
    
    .media-gallery {
      padding-right: 20px;
      @media screen and (max-width: 1199px) {
        padding-right: 0px;
        .el-col {
          width: 50%;
        }
      }

      .document-item {
        // background-color: #F0F0F0;
        font-size: 25px;
        display: flex;
        align-items: center;
        border-radius: 8px;
        border: 1.5px solid #f0f0f0;
        margin: 8px;
        display: flex;
        padding: 15px;
        @media screen and (max-width: 576px) {
          padding: 10px !important;
          font-size: 16px !important;
          .document-icon {
            margin-right: 10px !important;
          }
          .document-title {
            font-size: 12px !important;
            text-overflow: ellipsis;
            overflow: hidden;
          }
        }
        .document-icon {
          margin-right: 18px;
        }
        .document-title {
          font-size: 14px;
          white-space: nowrap;
          text-overflow: ellipsis;
          overflow: hidden;
        }
        cursor: pointer;
        &:hover, &.active {
          color: var(--primary-color);
          background-color: #DBC0CD !important;
          border: 1.5px solid #6B0036 !important;
          .document-icon {
            color: var(--primary-color);
          }
        }
      }
    }
    .placeholder {
        font-size: 16px;
        small {
            font-size: 72%;
            color: darken(#fff, 40%);
        }
    }
</style>