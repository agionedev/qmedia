<template>
  <div id="mediaMasterComponent">
    <!--Crud folders-->
    <crud :crud-data="import('modules/qmedia/_crud/folder')" type="onlyUpdate" ref="crudFolders"
          @created="refreshData" @updated="refreshData" @deleted="refreshData"
          :custom-data="customCrudData.folder" />

    <!--Crud Files-->
    <crud :crud-data="import('modules/qmedia/_crud/files')" type="onlyUpdate" ref="crudFiles"
          @created="refreshData" @updated="refreshData" @deleted="refreshData"
          :custom-data="customCrudData.file" />

    <!--Uploader-->
    <uploader ref="uploaderComponent" hide-file-list @added="uploadFiles"
              :max-files="50" :accept="accept" :max-file-size="maxFileSize" :ratio="ratio" />

    <!--Content-->
    <div class="relative-position" v-if="!onlyUpload">
      <!--Page Actions-->
      <div class="box box-auto-height q-mb-md">
        <page-actions :extra-actions="pageActions" title="Media" @search="val => {filter.search = val}" />
      </div>

      <!--Bread crumb-->
      <div class="box box-auto-height q-mb-md">
        <breadcrumb-component ref="breadcrumbComponent" :params="filter" @selected="setFolder" />
      </div>

      <!---Folders Files-->
      <div class="box box-auto-height q-mb-md">
        <file-list-component v-bind="fileListParams.folderFiles" @clickItem="setFolder" />
      </div>

      <!---Other Files-->
      <div class="box box-auto-height q-mb-md">
        <file-list-component v-bind="fileListParams.otherFiles" @selected="files => $emit('selected',files)" />
      </div>

      <!--inner loading-->
      <inner-loading :visible="loading" />
    </div>
  </div>
</template>
<script>
//Mixins
import zoneConfigMixing from 'modules/qmedia/_mixins/zoneConfigMixins';
//components
import uploader from 'modules/qsite/_components/master/uploader';
import breadcrumbComponent from 'modules/qmedia/_components/blocks/breadcrumb';
import fileListComponent from 'modules/qsite/_components/master/fileList';
import { eventBus } from 'src/plugins/utils';

export default {
  beforeUnmount() {
    eventBus.off('page.data.refresh');
  },
  mixins: [zoneConfigMixing],
  props: {
    disk: { default: null },
    allowSelect: { type: Number, default: 0 },
    maxFileSize: { type: Number, default: 0 },
    onlyUpload: { type: Boolean, defualt: false },
    ratio: { type: String, default: 'free' },
    accept: { default: false }
  },
  emits: ['selected', 'uploading', 'uploaded'],
  components: {
    breadcrumbComponent,
    fileListComponent,
    uploader
  },
  watch: {},
  mounted() {
    this.$nextTick(function() {
      this.init();
    });
  },
  data() {
    return {
      loading: false,
      listView: false,
      filter: {
        search: null,
        folderId: 0
      },
      filesComponent: null
    };
  },
  computed: {
    //page actions
    pageActions() {
      return [
        'search',
        {
          label: this.$tr('media.cms.newFolder'),
          props: {
            icon: 'fas fa-folder-plus',
            label: this.$tr('media.cms.newFolder'),
            round: false,
            rounded: true,
            padding: 'xs md',
            color: 'green'
          },
          action: () => this.$refs.crudFolders.create()
        },
        {
          label: this.$tr('media.cms.newFile'),
          props: {
            icon: 'fas fa-file-upload',
            label: this.$tr('media.cms.newFile'),
            round: false,
            rounded: true,
            padding: 'xs md',
            color: 'green'
          },
          action: () => this.$refs.uploaderComponent.pickFiles()
        }
      ];
    },
    //Item actions to file list
    itemFileListActions() {
      //Instance main actions
      let mainActions = [
        {
          label: this.$tr('isite.cms.label.edit'),
          icon: 'fas fa-pen',
          action: (item) => {
            if (item.isFolder) this.$refs.crudFolders.update(item);
            else this.$refs.crudFiles.update(item);
          }
        },
        {
          label: this.$tr('isite.cms.label.copyDisclosureLink'),
          icon: 'fas fa-copy',
          format: (item) => {
            return { vIf: !item.isFolder && item.url ? true : false };
          },
          action: (item) => this.$helper.copyToClipboard(item.url, 'isite.cms.messages.copyDisclosureLink')
        },
        {
          label: this.$tr('isite.cms.label.delete'),
          icon: 'fas fa-trash',
          action: (item) => {
            if (item.isFolder) this.$refs.crudFolders.delete(item);
            else this.$refs.crudFiles.delete(item);
          }
        }
      ];

      //Instance download action
      let downloadAction = {
        label: this.$tr('isite.cms.label.download'),
        vIf: this.$hasAccess('media.medias.download'),
        icon: 'fas fa-file-download',
        action: (item) => this.$helper.downloadFromURL(item.url)
      };

      //Response
      return {
        mainActions: mainActions,
        includeDownload: [downloadAction, ...mainActions]
      };
    },
    //Params to component files
    fileListParams() {
      return this.$clone({
        folderFiles: {
          ref: 'foldersFilesComponent',
          key: 'foldersContent',
          gridType: 'chip',
          icon: 'fas fa-folder-open',
          title: this.$trp('isite.cms.label.folder'),
          allowCounter: true,
          allowOrder: true,
          itemActions: this.allowSelect ? [] : this.itemFileListActions.mainActions,
          allowPagination: true,
          loadFiles: {
            apiRoute: 'apiRoutes.qmedia.files',
            requestParams: {
              take: 24,
              filter: {
                ...this.filter,
                isFolder: true,
                order: {
                  field: 'created_at',
                  way: 'desc'
                },
                disk: this.mediaDisk
              }
            }
          }
        },
        otherFiles: {
          ref: 'otherFilesComponent',
          key: 'otherFiles',
          icon: 'fas fa-photo-video',
          title: this.$trp('isite.cms.label.file'),
          allowCounter: true,
          allowOrder: true,
          itemActions: this.itemFileListActions.includeDownload,
          allowPagination: true,
          allowChangeView: true,
          allowSelect: this.allowSelect,
          loadFiles: {
            apiRoute: 'apiRoutes.qmedia.files',
            requestParams: {
              take: 24,
              filter: {
                ...this.filter,
                isFolder: false,
                order: {
                  field: 'created_at',
                  way: 'desc'
                },
                disk: this.mediaDisk,
                ...(this.accept ? { extension: this.acceptExtensions.withoutDot } : {})
              }
            }
          }
        }
      });
    },
    //Custom crud params
    customCrudData() {
      return {
        folder: {
          formLeft: {
            disk: { value: this.mediaDisk },
            parentId: {
              value: this.filter.folderId ? this.filter.folderId : '0',
              type: 'treeSelect',
              props: {
                label: this.$tr('isite.cms.form.parent'),
                clearable: false,
                options: [
                  { label: this.$tr('isite.cms.message.noValue'), value: '0', id: '0' }
                ]
              },
              loadOptions: {
                apiRoute: 'apiRoutes.qmedia.files',
                requestParams: { filter: { isFolder: true } },
                select: { label: 'filename', id: 'id', parentId: 'folderId' }
              }
            }
          }
        },
        file: {
          formLeft: {
            disk: { value: this.mediaDisk },
            folderId: {
              value: this.filter.folderId ? this.filter.folderId : '0',
              type: 'treeSelect',
              props: {
                label: this.$tr('isite.cms.label.folder'),
                clearable: false,
                options: [
                  { label: this.$tr('isite.cms.message.noValue'), value: '0', id: '0' }
                ]
              },
              loadOptions: {
                apiRoute: 'apiRoutes.qmedia.files',
                requestParams: { filter: { isFolder: true } },
                select: { label: 'filename', id: 'id', parentId: 'folderId' }
              }
            }
          }
        }
      };
    },
    //default disk
    mediaDisk() {
      return this.disk || this.$getSetting('media::filesystem');
    }
  },
  methods: {
    init() {
      eventBus.on('page.data.refresh', () => this.refreshData());
    },
    //upload files
    async uploadFiles(data) {
      this.loading = true; //Loading
      let filesUploaded = [];//Instance filesUploaded data
      var files = [data.file];//Get files and check be a array

      //Emit uploaded files
      this.$emit('uploading', this.$clone(files));

      //Upload files
      await Promise.all(files.map(async file => {
        //format request
        let fileData = new FormData();
        fileData.append('parent_id', this.filter.folderId || 0);
        fileData.append('disk', this.mediaDisk);
        fileData.append('file', file.file);

        //Request send file
        await this.$crud.post('apiRoutes.qmedia.files', fileData).then(response => {
          filesUploaded.push(response.data);
        });
      }));

      //Emit uploaded files
      this.$emit('uploaded', this.$clone(filesUploaded));

      //Last file uploaded
      if (data.final) {
        this.loading = false;
        this.refreshData();
      }
    },
    //Set folder
    setFolder(file) {
      if (file.isFolder) this.filter = { folderId: file.id, search: null };
    },
    //Refresh Data
    refreshData() {
      setTimeout(() => {
        if (this.$refs.breadcrumbComponent) this.$refs.breadcrumbComponent.getData(true);
        if (this.$refs.foldersFilesComponent) this.$refs.foldersFilesComponent.getData(true);
        if (this.$refs.otherFilesComponent) this.$refs.otherFilesComponent.getData(true);
      }, 100);
    },
    //Pick Files
    directUpload() {
      this.$refs.uploaderComponent.pickFiles();
    }
  }
};
</script>
<style lang="scss">
//#mediaMasterComponent
</style>
