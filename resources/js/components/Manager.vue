<template>
    <div ref="fileManagerContainer" id="filemanager-manager-container" class="p-3"  :class="cssFilemenagerContainer">
        <nav class="bg-grey-light rounded font-sans w-full m-4">
            <ol class="list-reset flex text-grey-dark">
                <li><span class="text-blue font-bold cursor-pointer" @click="goToFolderNav('/')">{{ __('Home') }}</span></li>
                <li v-if="pathsLength > 0"><span class="mx-2">/</span></li>
            

                <template v-for="(folder ,index) in path">
                    <template v-if="checkIsLastItem(index)">
                        <li  v-bind:key="index"><span href="#" class="text-blue">{{ folder.name }}</span></li>
                    </template>
                    <template v-else>
                        <li  v-bind:key="index"><span href="#" class="text-blue cursor-pointer font-bold" @click="goToFolder(folder.path)">{{ folder.name }}</span></li>
                        <li  v-bind:key="index+'_separator'"><span class="mx-2">/</span></li>
                    </template>
                </template>
            </ol>
        </nav>
        <transition name="fade">
            <div class="px-2 overflow-y-auto files" v-cloak>
                <div class="flex flex-wrap -mx-2">

                    <template v-if="files.error">
                        <div class="w-full text-lg text-center my-4">
                            {{ __('You don\'t have permissions to view this folder') }}
                        </div>
                    </template>

                    <template v-if="loading">
                        <div class="w-full text-lg text-center my-4">
                            <loading />
                        </div>
                    </template> 

                    

                    <template v-if="noFiles">
                        <div class="w-full text-lg text-center my-4">
                            {{ __('No files or folders in current directory') }}<br><br>
                            <button class="btn btn-default btn-danger" @click="removeDirectory">{{ __('Remove directory') }}</button>
                        </div>
                    </template>

                    <template v-if="!files.error" v-for="file in files">
                        <div :class="filemanagerClass" v-bind:key="file.id">
                            <template v-if="view == 'grid'">
                                <template v-if="file.type == 'file'">
                                    <ImageLoader :file="file" class="h-40" @missing="(value) => missing = value" v-on:showInfo="showInfo" />
                                </template>
                                <template v-if="file.type == 'dir'">
                                    <Folder :file="file" class="h-40" :class="{'loading': loadingInfo}" v-on:goToFolderEvent="goToFolder" />
                                </template>
                            </template>

                            <template v-else>
                                <template v-if="file.type == 'file'">
                                    <ImageLoader :file="file" :view="view" class="h-8" :class="{'loading': loadingInfo}" @missing="(value) => missing = value" v-on:showInfo="showInfo" />
                                </template>
                                <template v-if="file.type == 'dir'">
                                    <Folder :file="file" :view="view" class="h-8" :class="{'loading': loadingInfo}" v-on:goToFolderEvent="goToFolder" />
                                </template>
                            </template>
                            
                        </div>
                    </template>


                </div>
                 

            </div>

            
        </transition>
        
    </div>
</template>

<script>
import _ from 'lodash';
import filesize from 'filesize';
import MD5 from '../tools/md5';
import api from '../api';
import ImageLoader from '../modules/ImageLoader';
import Folder from '../modules/Folder';
import Loading from './Loading';
let arrayFiles = [];

export default {
    components: {
        ImageLoader: ImageLoader,
        Folder: Folder,
        loading: Loading,
    },

    props: {
        files: {
            default: function() {
                return [];
            },
            required: true,
        },
        path: {
            default: function() {
                return [];
            },
            required: true,
        },
        current: {
            type: String,
            default: '/',
            required: true,
        },
        noFiles: {
            type: Boolean,
            default: false,
            required: true,
        },
        loading: {
            type: Boolean,
            default: false,
            required: true,
        },
        popupLoaded: {
            type: Boolean,
            defalut: false,
            required: false,
        },
        view: {
            type: String,
            default: 'grid',
            required: false,
        },
    },

    data: () => ({
        info: {},
        activeInfo: false,
        eventsLoaded: false,
        cssDragAndDrop: null,
        filesToUpload: [],
        loadingInfo: false,
    }),

    methods: {
        goToFolder(path) {
            this.$emit('goToFolderManager', path);
        },

        goToFolderNav(path) {
            this.$emit('goToFolderManagerNav', path);
        },

        checkIsLastItem(index) {
            return _.size(this.path) == parseInt(index) + 1 ? true : false;
        },

        removeDirectory() {
            return api.removeDirectory(this.current).then(result => {
                if (result == true) {
                    this.$toasted.show(this.__('Folder removed successfully'), { type: 'success' });
                    this.$emit('goToFolderManager', '/');
                } else {
                    this.$toasted.show(
                        this.__('Error removing the folder. Please check permissions'),
                        { type: 'error' }
                    );
                }
            });
        },

        showInfo(file) {
            file.loading = true;
            return api
                .getInfo(file.path)
                .then(result => {
                    this.$emit('showInfoItem', result);
                    file.loading = false;
                })
                .catch(() => {
                    file.loading = false;
                    this.$toasted.show(this.__('Error opening the file. Check your permissions'), {
                        type: 'error',
                    });
                });
        },

        closePreview() {
            this.activeInfo = false;
            this.info = {};
        },

        refresh() {
            this.$emit('refresh');
        },

        selectFile(file) {
            this.$emit('selectFile', file);
        },

        setDragAndDropEvents() {
            let self = this;

            let filemanagerContainer = document.querySelector('#filemanager-manager-container');

            filemanagerContainer.addEventListener('dragenter', function(e) {
                e.preventDefault();
                self.cssDragAndDrop = 'inside';
            });

            filemanagerContainer.addEventListener('dragleave', function(e) {
                e.preventDefault();
                self.cssDragAndDrop = 'outside';
            });

            filemanagerContainer.addEventListener('dragover', function(e) {
                e.preventDefault();
                self.cssDragAndDrop = 'inside';
            });

            filemanagerContainer.addEventListener('drop', function(e) {
                e.preventDefault();
                self.cssDragAndDrop = 'drop';

                let files = e.dataTransfer.files;
                self.uploadFiles(files);
            });
        },

        uploadFiles(files) {
            Array.from(files).forEach(file => {
                arrayFiles.push({
                    id: MD5(file.name),
                    preview: this.getPreview(file),
                    type: file.type,
                    name: file.name,
                    size: filesize(file.size),
                    upload: true,
                    progress: 0,
                    error: false,
                    file: file,
                });
            });

            this.filesToUpload = arrayFiles;

            this.$emit('uploadFiles', arrayFiles);
        },

        getPreview(file) {
            if (this.isImage(file)) {
                return URL.createObjectURL(file);
            }

            return file.name.split('.').pop();
        },

        isImage(file) {
            return file.type.includes('image'); //returns true or false
        },
    },

    updated: function() {
        //
    },

    mounted() {
        if (!this.eventsLoaded) {
            this.$nextTick(function() {
                setTimeout(() => {
                    this.setDragAndDropEvents();
                    this.eventsLoaded = true;
                }, 500);
            });
        }
    },

    computed: {
        pathsLength() {
            return _.size(this.path);
        },

        filesCount() {
            return _.size(this.files);
        },

        filemanagerClass() {
            if (this.view == 'grid') {
                return 'w-1/6 h-40  px-2 mb-3';
            } else {
                return 'w-full h-8  px-2 mb-3';
            }
        },

        filemanagerIconClass() {
            if (this.view == 'grid') {
                return 'h-40';
            } else {
                return 'h-8';
            }
        },
        cssFilemenagerContainer() {
            if (this.cssDragAndDrop == 'inside') {
                return 'bg-20';
            }

            if (this.cssDragAndDrop == 'outside') {
                return '';
            }
            return '';
        },
    },
};
</script>

<style>
/* Scoped Styles */
.w-1\/8 {
    width: 12.5%;
}
.w-40 {
    width: 10rem;
}

.h-40 {
    height: 10rem;
}

.obfit-cover {
    object-fit: cover;
}

.files {
    max-height: 60vh;
}
</style>
