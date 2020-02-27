<template>
    <default-field :field="field" :errors="errors" :full-width-content="true">
        <template slot="field">
            <div class="rounded-lg form-control auto-height">
                <ckeditor
                    :editor="editor"
                    :config="editorConfig"
                    :id="field.name"
                    :class="errorClasses"
                    :placeholder="field.name"
                    v-model="value"
                    @ready="setEditorInitialValue"
                />
            </div>
        </template>
    </default-field>
</template>

<script>
import { FormField, HandlesValidationErrors } from 'laravel-nova'
import CKEditor from '@ckeditor/ckeditor5-vue'
import ClassicEditor from '@ckeditor/ckeditor5-build-classic'
import NovaCKEditor5UploadAdapter from '../ckeditor5/upload-adapter'

export default {
    mixins: [FormField, HandlesValidationErrors],

    components: {
        ckeditor: CKEditor.component
    },

    props: ['resourceName', 'resourceId', 'field'],

    data () {
        return {
            editor: ClassicEditor,
            defaultEditorConfig: {
                nova: {
                    resourceName: this.resourceName,
                    field: this.field,
                    draftId: uuidv4()
                },
                language: 'en',
                toolbar: this.field.options.toolbar,
                heading: this.field.options.heading,
                image: this.field.options.image,
                fontFamily: this.field.options.fontFamily,
                extraPlugins: [
                    this.createUploadAdapterPlugin
                ],
                link: this.field.options.link
            }
        }
    },

    computed: {
        draftId: function () {
            return this.defaultEditorConfig.nova.draftId
        },
        editorConfig: function() {
            let editorConfig = this.defaultEditorConfig

            if (! editorConfig.nova.field.withFiles) {
                editorConfig.removePlugins = [
                    'Image',
                    'ImageToolbar',
                    'ImageCaption',
                    'ImageStyle',
                    'ImageTextAlternate',
                    'ImageUpload'
                ]
                editorConfig.image = {}
                editorConfig.extraPlugins = []
            }

            return editorConfig
        }
    },

    beforeDestroy() {
        this.cleanUp()
    },

    methods: {
        /**
         * Set the initial, internal value for the field.
         */
        setInitialValue() {
            //
        },

        /**
         * Set the editor initial, internal value for the field when the editor is ready.
         */
        setEditorInitialValue(editor) {
            this.value = this.field.value || '';




            // elfinder folder hash of the destination folder to be uploaded in this CKeditor 5
            const uploadTargetHash = 'l1_Lw';

            // elFinder connector URL
            const connectorUrl = '/elfinder/connector';

            const ckf = editor.commands.get('ckfinder'),
                fileRepo = editor.plugins.get('FileRepository'),
                ntf = editor.plugins.get('Notification'),
                i18 = editor.locale.t,
                // Insert images to editor window
                insertImages = urls => {
                    const imgCmd = editor.commands.get('imageUpload');
                    if (!imgCmd.isEnabled) {
                        ntf.showWarning(i18('Could not insert image at the current position.'), {
                            title: i18('Inserting image failed'),
                            namespace: 'ckfinder'
                        });
                        return;
                    }
                    editor.execute('imageInsert', { source: urls });
                },
                // To get elFinder instance
                getfm = open => {
                    return new Promise((resolve, reject) => {
                        // Execute when the elFinder instance is created
                        const done = () => {
                            if (open) {
                                // request to open folder specify
                                if (!Object.keys(_fm.files()).length) {
                                    // when initial request
                                    _fm.one('open', () => {
                                        _fm.file(open)? resolve(_fm) : reject(_fm, 'errFolderNotFound');
                                    });
                                } else {
                                    // elFinder has already been initialized
                                    new Promise((res, rej) => {
                                        if (_fm.file(open)) {
                                            res();
                                        } else {
                                            // To acquire target folder information
                                            _fm.request({cmd: 'parents', target: open}).done(e =>{
                                                _fm.file(open)? res() : rej();
                                            }).fail(() => {
                                                rej();
                                            });
                                        }
                                    }).then(() => {
                                        // Open folder after folder information is acquired
                                        _fm.exec('open', open).done(() => {
                                            resolve(_fm);
                                        }).fail(err => {
                                            reject(_fm, err? err : 'errFolderNotFound');
                                        });
                                    }).catch((err) => {
                                        reject(_fm, err? err : 'errFolderNotFound');
                                    });
                                }
                            } else {
                                // show elFinder manager only
                                resolve(_fm);
                            }
                        };

                        // Check elFinder instance
                        if (_fm) {
                            // elFinder instance has already been created
                            done();
                        } else {
                            // To create elFinder instance
                            _fm = $('<div/>').dialogelfinder({
                                // dialog title
                                title : 'File Manager',
                                // connector URL
                                url : connectorUrl,
                                // start folder setting
                                startPathHash : open? open : void(0),
                                // Set to do not use browser history to un-use location.hash
                                useBrowserHistory : false,
                                // Disable auto open
                                autoOpen : false,
                                // elFinder dialog width
                                width : '80%',
                                // set getfile command options
                                commandsOptions : {
                                    getfile: {
                                        oncomplete : 'close',
                                        multiple : true
                                    }
                                },
                                // Insert in CKEditor when choosing files
                                getFileCallback : (files, fm) => {
                                    let imgs = [];
                                    fm.getUI('cwd').trigger('unselectall');
                                    $.each(files, function(i, f) {
                                        if (f && f.mime.match(/^image\//i)) {
                                            imgs.push(fm.convAbsUrl(f.url));
                                        } else {
                                            editor.execute('link', fm.convAbsUrl(f.url));
                                        }
                                    });
                                    if (imgs.length) {
                                        insertImages(imgs);
                                    }
                                }
                            }).elfinder('instance');
                            done();
                        }
                    });
                };

            // elFinder instance
            let _fm;

            if (ckf) {
                // Take over ckfinder execute()
                ckf.execute = () => {
                    getfm().then(fm => {
                        fm.getUI().dialogelfinder('open');
                    });
                };
            }

            // Make uploader
            const uploder = function(loader) {
                this.upload = function() {
                    return new Promise(function(resolve, reject) {
                        getfm(uploadTargetHash).then(fm => {
                            let fmNode = fm.getUI();
                            fmNode.dialogelfinder('open');
                            fm.exec('upload', {files: [loader.file], target: uploadTargetHash}, void(0), uploadTargetHash)
                                .done(data => {
                                    if (data.added && data.added.length) {
                                        fm.url(data.added[0].hash, { async: true }).done(function(url) {
                                            resolve({
                                                'default': fm.convAbsUrl(url)
                                            });
                                            fmNode.dialogelfinder('close');
                                        }).fail(function() {
                                            reject('errFileNotFound');
                                        });
                                    } else {
                                        reject(fm.i18n(data.error? data.error : 'errUpload'));
                                        fmNode.dialogelfinder('close');
                                    }
                                })
                                .fail(err => {
                                    const error = fm.parseError(err);
                                    reject(fm.i18n(error? (error === 'userabort'? 'errAbort' : error) : 'errUploadNoFiles'));
                                });
                        }).catch((fm, err) => {
                            const error = fm.parseError(err);
                            reject(fm.i18n(error? (error === 'userabort'? 'errAbort' : error) : 'errUploadNoFiles'));
                        });
                    });
                };
                this.abort = function() {
                    _fm && _fm.getUI().trigger('uploadabort');
                };
            };
            // Set up image uploader
            fileRepo.createUploadAdapter = loader => {
                return new uploder(loader);
            };
        },

        /**
         * Fill the given FormData object with the field's internal value.
         */
        fill(formData) {
            formData.append(this.field.attribute, this.value || '')
            formData.append(this.field.attribute + 'DraftId', this.draftId);
        },

        /**
         * Update the field's internal value.
         */
        handleChange(value) {
            this.value = value
        },

        /**
        * Create CKEditor upload adapter plugin.
        */
        createUploadAdapterPlugin(editor) {
            let novaConfig = editor.config.get('nova')

            editor.plugins.get('FileRepository').createUploadAdapter = (loader) => {
                return new NovaCKEditor5UploadAdapter(loader, novaConfig.resourceName, novaConfig.field, novaConfig.draftId)
            }
        },

        /**
         * Purge pending attachments for the draft
         */
        cleanUp() {
            if (this.field.withFiles) {
                Nova.request()
                    .delete(
                        `/nova-vendor/ckeditor5-classic/${this.resourceName}/attachments/${this.field.attribute}/${this.draftId}`
                    )
                    .then(response => console.log(response))
                    .catch(error => {})
            }
        }

    }
}

function uuidv4() {
    return ([1e7] + -1e3 + -4e3 + -8e3 + -1e11).replace(/[018]/g, c =>
        (c ^ (crypto.getRandomValues(new Uint8Array(1))[0] & (15 >> (c / 4)))).toString(16)
    )
}
</script>
