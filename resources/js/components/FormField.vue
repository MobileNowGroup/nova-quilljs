<template>
    <DefaultField
        :field="field"
        :errors="errors"
    >
        <template #field>
            <quill-editor
                :style="css"
                v-model:value="value"
                ref="myQuillEditor"
                :options="editorOption"
                @blur="onEditorBlur($event)"
                @focus="onEditorFocus($event)"
                @ready="onEditorReady($event)"
            ></quill-editor>
        </template>
    </DefaultField>
</template>

<script>
import {FormField, HandlesValidationErrors} from "laravel-nova";
import {quillEditor, Quill} from 'vue3-quill'
import BlotFormatter from "quill-blot-formatter";
import {ImageExtend, QuillWatch} from "quill-image-extend-module";
import Tooltip from "quill/ui/tooltip";
import {CustomImageSpec} from "../../quilljs/CustomImageSpec";
import "quill/dist/quill.core.css";
import "quill/dist/quill.snow.css";
import "quill/dist/quill.bubble.css";
import {VideoHandler } from "quill-upload";

Quill.register({
    "modules/ImageExtend": ImageExtend,
    "modules/blotFormatter": BlotFormatter,
    "ui/tooltip": Tooltip,
    "modules/videoHandler": VideoHandler,
});
const Delta = Quill.import('delta');

const _onUpload = function (fd, resolve) {
  const xhr = new XMLHttpRequest();
  xhr.open("POST", "/nova-vendor/quill-video-upload");

  xhr.setRequestHeader(
      "X-CSRF-TOKEN",
      document.head.querySelector('meta[name="csrf-token"]').content
  );

  xhr.onload = () => {
    if (xhr.status === 200) {
      const response = JSON.parse(xhr.responseText);

      resolve(response.url); // Must resolve as a link to the image
    }
  };

  xhr.send(fd);
};

export default {
    mixins: [FormField, HandlesValidationErrors],
    components: {
        quillEditor,
    },
    props: ["resourceName", "resourceId", "field"],
    data() {
        return {
            value: this.field.value,
            persisted: [],
            toolbarTips: this.field.tooltip,
            editorOption: {
                placeholder: this.field.placeholder,
                modules: {
                    blotFormatter: {
                        specs: [CustomImageSpec],
                    },
                    ImageExtend: {
                        loading: true,
                        size: this.field.maxFileSize ? this.field.maxFileSize : 2,
                        name: "attachment",
                        action: `/nova-vendor/quilljs/${this.resourceName}/upload/${this.field.attribute.split(this.field.split)[0]}`,
                        response: (res) => {
                            return res.url;
                        },
                        headers: (xhr) => {
                            xhr.setRequestHeader(
                                "X-CSRF-TOKEN",
                                document.head.querySelector('meta[name="csrf-token"]').content
                            );
                        },
                        sizeError: () => {
                            this.$toasted.show(
                                `Image size exceeds ${
                                    this.field.maxFileSize ? this.field.maxFileSize : 2
                                }MB`,
                                {type: "error"}
                            );
                        },
                        change: (xhr, formData) => {
                            const draftId = this._uuid()
                            formData.append("draftId", draftId)
                            this.persisted.push(draftId)

                        },
                    },
                    videoHandler: {
                      upload: file => {
                        // return a Promise that resolves in a link to the uploaded image
                        return new Promise((resolve, reject) => {
                          const fd = new FormData();
                          fd.append("file", file);
                          fd.append("fileName", file.name);

                          return _onUpload(fd, resolve);
                        });
                      }
                    },
                    toolbar: {
                        container: this.field.options
                    },
                },
            },
        };
    },
    methods: {
        _uuid() {
            var d = Date.now();
            if (
                typeof performance !== "undefined" &&
                typeof performance.now === "function"
            ) {
                d += performance.now(); //use high-precision timer if available
            }
            return "xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx".replace(/[xy]/g, function (
                c
            ) {
                var r = (d + Math.random() * 16) % 16 | 0;
                d = Math.floor(d / 16);
                return (c === "x" ? r : (r & 0x3) | 0x8).toString(16);
            });
        },
        /*
         * Set the initial, internal value for the field.
         */
        setInitialValue() {
            this.value = this.field.value || "";
        },

        /**
         * Fill the given FormData object with the field's internal value.
         */
        fill(formData) {
            formData.append(this.field.attribute, this.value || "");
            if (!formData.has('persisted')) {
                formData.append('persisted', JSON.stringify(this.persisted));
            }
        },

        /**
         * Update the field's internal value.
         */
        handleChange(value) {
            this.value = value;
        },

        onEditorBlur(quill) {
            // console.log("editor blur!", quill);
        },
        onEditorFocus(quill) {
            // console.log("editor focus!", quill);
        },
        onEditorReady(quill) {
            // console.log("editor ready!", quill);
        },
        onEditorChange({quill, html, text}) {
            this.content = html;
        },
        autotip() {
            if (this.toolbarTips) {
                for (let item of this.toolbarTips) {
                    let tip = document.querySelector(".quill-editor " + item.Choice);
                    if (!tip) continue;
                    tip.setAttribute("title", item.title);
                }
            }
        }
    },
    computed: {
        editor() {
            return this.$refs.myQuillEditor.quill;
        },
        css() {
            return {
                height: this.field.height + 41 + "px",
                "padding-bottom": this.field.paddingBottom + 40 + "px",
            };
        },
    },
    mounted() {
        this.autotip()
    },
};
</script>

<style>
.ql-editor p {
    margin-top: 18px;
    font-size: 18px;
}

.ql-video {
    width: 800px;
    height: 450px;
}
</style>
