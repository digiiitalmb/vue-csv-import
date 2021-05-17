<template>
    <div class="vue-csv-uploader">
        <div class="form">
            <div class="vue-csv-uploader-part-one">


                <div class="form-check form-group csv-import-checkbox" v-if="headers === null">
                    <slot name="hasHeaders" :headers="hasHeaders" :toggle="toggleHasHeaders">
                        <input
                            :class="checkboxClass" type="checkbox"
                            :id="makeId('hasHeaders')"
                            :value="hasHeaders"
                            @change="toggleHasHeaders"
                        />
                        <label class="form-check-label" :for="makeId('hasHeaders')">
                            File Has Headers
                        </label>
                    </slot>
                </div>


                <div class="card card-default">
                        <div class="card-body">

                            <div class="form-group row mb-0">
                                <label class="col-md-4 col-form-label text-md-right">CSV File</label>

                                <div class="col-md-6">
                                    <div class="input-group mb-0">

                                          <div class="custom-file csv-import-file">
                                            
                                            <label class="custom-file-label form-control" for="inputGroupFile01">Choose file</label>
                                            <input 
                                            type="file" 
                                            id="inputGroupFile01" 
                                            aria-describedby="inputGroupFileAddon01" 
                                            ref="csv"
                                            @change.prevent="validFileMimeType"
                                            class="custom-file-input file-select"
                                            name="csv">
                                           
                                          </div>

                                    </div>
                                </div>
                            </div>

                            <slot name="error" v-if="showErrorMessage">
                                <div class="invalid-feedback d-block">
                                    File type is invalid
                                </div>
                            </slot>

                        </div>
                    </div> 


            </div>
            <div class="vue-csv-uploader-part-two">
                <div class="vue-csv-mapping" v-if="sample">
                    <table :class="tableClass">
                        <slot name="thead">
                            <thead>
                                <tr>
                                    <th><span class="small text-uppercase font-weight-bold text-muted">Field</span></th>
                                    <th><span class="small text-uppercase font-weight-bold text-muted">CSV Column</span></th>
                                </tr>
                            </thead>
                        </slot>
                        <tbody>
                            <tr v-for="(field, key) in fieldsToMap" :key="key">
                                <td><a>{{ field.label }}</a></td>
                                <td>
                                    <div class="py-1">
                                    <v-select
                                        placeholder="Select column"
                                        :name="`csv_uploader_map_${key}`"
                                        v-model="map[field.key]"
                                        :options="selectOptions"
                                    >                           
                                    </v-select>
                                    </div>
                                </td> 
                            </tr>
                        </tbody>
                    </table>
                    <div class="form-group" v-if="url">
                        <slot name="submit" :submit="submit">
                            <input type="submit" :class="buttonClass" @click.prevent="submit" :value="submitBtnText"/>
                        </slot>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>
<!--
  <option :value="null" v-if="canIgnore">Ignore</option>
                                        <option v-for="(column, key) in firstRow" :key="key" :value="key">
                                            {{ column }}
                                        </option>
-->
<script>
import {drop, every, forEach, get, isArray, map, set} from "lodash";
import axios from "axios";
import Papa from "papaparse";
import mimeTypes from "mime-types";
import vSelect from 'vue-select';

vSelect.props.components.default = () => ({
  Deselect: {
    render: createElement => createElement('i', {
        class: {
            'far fa-times mr-1':true
        }
    }),
  },
  OpenIndicator: {
    render: createElement => createElement('i', {
        class: {
            'far fa-angle-down mr-0':true
        }
    }),
  },
});
 
export default {
    components: { 
        'v-select':vSelect 
    },
    props: {
        value: Array,
        url: {
            type: String,
        },
        mapFields: {
            required: true,
        },
        callback: {
            type: Function,
            default: () => ({}),
        },
        catch: {
            type: Function,
            default: () => ({}),
        },
        finally: {
            type: Function,
            default: () => ({}),
        },
        parseConfig: {
            type: Object,
            default() {
                return {};
            },
        },
        headers: {
            default: null,
        },
        loadBtnText: {
            type: String,
            default: "Next",
        },
        submitBtnText: {
            type: String,
            default: "Submit",
        },
        autoMatchFields: {
            type: Boolean,
            default: false,
        },
        autoMatchIgnoreCase: {
            type: Boolean,
            default: false,
        },
        tableClass: {
            type: String,
            default: "table",
        },
        checkboxClass: {
            type: String,
            default: "form-check-input",
        },
        buttonClass: {
            type: String,
            default: "btn btn-primary",
        },
        inputClass: {
            type: String,
            default: "form-control-file",
        },
        validation: {
            type: Boolean,
            default: true,
        },
        fileMimeTypes: {
            type: Array,
            default: () => {
                return ["text/csv", "text/x-csv", "application/vnd.ms-excel", "text/plain"];
            },
        },
        tableSelectClass: {
            type: String,
            default: "form-control",
        },
        canIgnore: {
            type: Boolean,
            default: false,
        },
    },

    data: () => ({
        form: {
            csv: null,
        },
        fieldsToMap: [],
        map: {},
        hasHeaders: true,
        csv: null,
        sample: null,
        isValidFileMimeType: false,
        fileSelected: false,
    }),

    created() {
        this.hasHeaders = this.headers;

        if (isArray(this.mapFields)) {
            this.fieldsToMap = map(this.mapFields, (item) => {
                return {
                    key: item,
                    label: item,
                };
            });
        } else {
            this.fieldsToMap = map(this.mapFields, (label, key) => {
                return {
                    key: key,
                    label: label,
                };
            });
        }
    },

    methods: {
        submit() {
            const _this = this;
            this.form.csv = this.buildMappedCsv();
            this.$emit("input", this.form.csv);

            if (this.url) {
                axios
                    .post(this.url, this.form)
                    .then((response) => {
                        _this.callback(response);
                    })
                    .catch((response) => {
                        _this.catch(response);
                    })
                    .finally((response) => {
                        _this.finally(response);
                    });
            } else {
                _this.callback(this.form.csv);
            }
        },
        buildMappedCsv() {
            const _this = this;

            let csv = this.hasHeaders ? drop(this.csv) : this.csv;

            return map(csv, (row) => {
                let newRow = {};

                forEach(_this.map, (column, field) => {
                    set(newRow, field, get(row, column));
                });

                return newRow;
            });
        },
        validFileMimeType() {
            let file = this.$refs.csv.files[0];
            const mimeType = file.type === "" ? mimeTypes.lookup(file.name) : file.type;

            if (file) {
                this.fileSelected = true;
                this.isValidFileMimeType = this.validation ? this.validateMimeType(mimeType) : true;
                this.load();
                console.log('attempt load');
            } else {
                this.isValidFileMimeType = !this.validation;
                this.fileSelected = false;
            }
        },
        validateMimeType(type) {
            return this.fileMimeTypes.indexOf(type) > -1;
        },
        load() {
            const _this = this;

            console.log('log');
            
            this.readFile((output) => {
                _this.sample = get(Papa.parse(output, {preview: 2, skipEmptyLines: true}), "data");
                _this.csv = get(Papa.parse(output, {skipEmptyLines: true}), "data");
            });
        },
        readFile(callback) {
            let file = this.$refs.csv.files[0];

            if (file) {
                let reader = new FileReader();
                reader.readAsText(file, "UTF-8");
                reader.onload = function (evt) {
                    callback(evt.target.result);
                };
                reader.onerror = function () {
                };
            }
        },
        toggleHasHeaders() {
            this.hasHeaders = !this.hasHeaders;
        },
        makeId(id) {
            return `${id}${this._uid}`;
        },
    },
    watch: {
        map: {
            deep: true,
            handler: function (newVal) {
                if (!this.url) {
                    let hasAllKeys = Array.isArray(this.mapFields)
                        ? every(this.mapFields, function (item) {
                            return Object.prototype.hasOwnProperty.call(newVal, item);
                        })
                        : every(this.mapFields, function (item, key) {
                            return Object.prototype.hasOwnProperty.call(newVal, key);
                        });

                    if (hasAllKeys) {
                        this.submit();
                    }
                }
            },
        },
        sample(newVal) {
            if (this.autoMatchFields) {
                if (newVal !== null) {
                    this.fieldsToMap.forEach((field) => {
                        newVal[0].forEach((columnName, index) => {
                            if (this.autoMatchIgnoreCase === true) {
                                if (field.label.toLowerCase().trim() === columnName.toLowerCase().trim()) {
                                    this.$set(this.map, field.key, index);
                                }
                            } else {
                                if (field.label.trim() === columnName.trim()) {
                                    this.$set(this.map, field.key, index);
                                }
                            }
                        });
                    });
                }
            }
        },
    },
    computed: {
        selectOptions() {

            var options = get(this, "sample.0");
            console.log(options);

            return [{label: 'Canada', code: 'ca'}];
        },
        firstRow() {
            return get(this, "sample.0");
        },
        showErrorMessage() {
            return this.fileSelected && !this.isValidFileMimeType;
        },
        disabledNextButton() {
            return !this.isValidFileMimeType;
        },
    },
};
</script>
