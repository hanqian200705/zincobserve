<!-- Copyright 2023 Zinc Labs Inc.

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU Affero General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU Affero General Public License for more details.

You should have received a copy of the GNU Affero General Public License
along with this program.  If not, see <http://www.gnu.org/licenses/>.
-->

<template>
  <div class="full-width">
    <div class="row items-center no-wrap q-mx-md q-my-sm">
      <div class="flex items-center" data-test="add-alert-title">
        <div class="flex justify-center items-center q-mr-md cursor-pointer" style="
            border: 1.5px solid;
            border-radius: 50%;
            width: 22px;
            height: 22px;
          " title="Go Back" @click="router.back()">
          <q-icon name="arrow_back_ios_new" size="14px" />
        </div>
        <div v-if="beingUpdated" class="text-h6">
          {{ t("alerts.updateTitle") }}
        </div>
        <div v-else class="text-h6">{{ t("alerts.addTitle") }}</div>
      </div>
    </div>

    <q-separator />
    <div ref="addAlertFormRef" class="q-px-lg q-my-md" style="
        max-height: calc(100vh - 138px);
        overflow: auto;
        scroll-behavior: smooth;
      ">
      <div class="row justify-start items-start" style="width: 1024px">
        <div style="width: calc(100% - 401px)">
          <q-form class="add-alert-form" ref="addAlertForm" @submit="onSubmit">
            <div class="flex justify-start items-center q-pb-sm q-col-gutter-md">
              <div class="alert-name-input o2-input" style="padding-top: 12px">
                <q-input data-test="add-alert-name-input" v-model="formData.name" :label="t('alerts.name') + ' *'"
                  color="input-border" bg-color="input-bg" class="showLabelOnTop" stack-label outlined filled dense
                  v-bind:readonly="beingUpdated" v-bind:disable="beingUpdated"
                  :rules="[(val: any) => !!val.trim() || 'Field is required!']" tabindex="0" style="min-width: 220px" />
              </div>
              <div class="alert-stream-type o2-input" style="padding-top: 0">
                <q-select v-model="formData.stream_type" :options="streamTypes" :label="t('alerts.streamType') + ' *'"
                  :popup-content-style="{ textTransform: 'lowercase' }" color="input-border" bg-color="input-bg"
                  class="q-py-sm showLabelOnTop no-case" stack-label outlined filled dense v-bind:readonly="beingUpdated"
                  v-bind:disable="beingUpdated" @update:model-value="updateStreams()"
                  :rules="[(val: any) => !!val || 'Field is required!']" style="min-width: 120px" />
              </div>
              <div class="o2-input" style="padding-top: 0">
                <q-select data-test="add-alert-stream-select" v-model="formData.stream_name" :options="filteredStreams"
                  :label="t('alerts.stream_name') + ' *'" :loading="isFetchingStreams"
                  :popup-content-style="{ textTransform: 'lowercase' }" color="input-border" bg-color="input-bg"
                  class="q-py-sm showLabelOnTop no-case" stack-label outlined filled dense v-bind:readonly="beingUpdated"
                  v-bind:disable="beingUpdated" :input-debounce="400" @filter="filterStreams"
                  @update:model-value="updateStreamFields(formData.stream_name)"
                  :rules="[(val: any) => !!val || 'Field is required!']" style="min-width: 220px" />
              </div>
            </div>
            <div class="q-gutter-sm">
              <q-radio data-test="add-alert-scheduled-alert-radio" v-bind:readonly="beingUpdated"
                v-bind:disable="beingUpdated" v-model="formData.is_real_time" :checked="formData.is_real_time" val="false"
                :label="t('alerts.scheduled')" class="q-ml-none" />
              <q-radio data-test="add-alert-realtime-alert-radio" v-bind:readonly="beingUpdated"
                v-bind:disable="beingUpdated" v-model="formData.is_real_time" :checked="!formData.is_real_time" val="true"
                :label="t('alerts.realTime')" class="q-ml-none" />
            </div>
            <div v-if="formData.is_real_time === 'true'" class="q-py-sm showLabelOnTop text-bold text-h7"
              data-test="add-alert-query-input-title">
              <real-time-alert :columns="filteredColumns" :conditions="formData.query_condition.conditions"
                @field:add="addField" @field:remove="removeField" @input:update="onInputUpdate" />
            </div>
            <div v-else>
              <scheduled-alert ref="scheduledAlertRef" :columns="filteredColumns"
                :conditions="formData.query_condition.conditions" v-model:trigger="formData.trigger_condition"
                v-model:sql="formData.query_condition.sql" v-model:query_type="formData.query_condition.type"
                v-model:aggregation="formData.query_condition.aggregation"
                v-model:isAggregationEnabled="isAggregationEnabled" @field:add="addField" @field:remove="removeField"
                @input:update="onInputUpdate" class="q-mt-sm" />
            </div>

            <div class="col-12 flex justify-start items-center q-mt-xs">
              <div class="q-py-sm showLabelOnTop text-bold text-h7 q-pb-md" data-test="add-alert-delay-title"
                style="width: 180px">
                {{ t("alerts.silenceNotification") + " *" }}
              </div>
              <div style="min-height: 58px">
                <div class="col-8 row justify-left align-center q-gutter-sm">
                  <div class="flex items-center" style="border: 1px solid rgba(0, 0, 0, 0.05)">
                    <div style="width: 87px; margin-left: 0 !important" class="silence-notification-input">
                      <q-input data-test="add-alert-delay-input" v-model="formData.trigger_condition.silence"
                        type="number" dense filled min="1" style="background: none" />
                    </div>
                    <div style="
                        min-width: 90px;
                        margin-left: 0 !important;
                        background: #f2f2f2;
                        height: 40px;
                      " class="flex justify-center items-center">
                      {{ t("alerts.minutes") }}
                    </div>
                  </div>
                </div>
                <div v-if="formData.trigger_condition.silence < 1" class="text-red-8 q-pt-xs"
                  style="font-size: 11px; line-height: 12px">
                  Field is required!
                </div>
              </div>
            </div>

            <div class="o2-input flex justify-start items-center">
              <div class="text-bold q-pb-sm" style="width: 180px">
                {{ t("alerts.destination") + " *" }}
              </div>
              <q-select data-test="add-alert-destination-select" v-model="formData.destinations"
                :options="getFormattedDestinations" color="input-border" bg-color="input-bg q-mt-sm" class="no-case"
                stack-label outlined filled dense multiple :rules="[(val: any) => !!val || 'Field is required!']"
                style="width: 250px">
                <template v-slot:option="option">
                  <q-list dense>
                    <q-item tag="label">
                      <q-item-section avatar>
                        <q-checkbox size="xs" dense v-model="formData.destinations" :val="option.opt" />
                      </q-item-section>
                      <q-item-section>
                        <q-item-label class="ellipsis">{{ option.opt }}
                        </q-item-label>
                      </q-item-section>
                    </q-item>
                  </q-list>
                </template>
              </q-select>
            </div>

            <div class="q-mt-md">
              <div class="text-bold">{{ t("alerts.additionalVariables") }}</div>
              <variables-input class="o2-input" :variables="formData.context_attributes" @add:variable="addVariable"
                @remove:variable="removeVariable" />
            </div>

            <div class="o2-input">
              <q-input data-test="add-alert-description-input" v-model="formData.description"
                :label="t('alerts.description')" color="input-border" bg-color="input-bg" class="showLabelOnTop q-mb-sm"
                stack-label outlined filled dense tabindex="0" style="width: 550px" />

              <q-input data-test="add-alert-row-input" v-model="formData.row_template" :label="t('alerts.row')"
                color="input-border" bg-color="input-bg" class="showLabelOnTop" stack-label outlined filled dense
                tabindex="0" style="width: 550px" />
            </div>

            <div class="flex justify-start q-mt-lg">
              <q-btn data-test="add-alert-cancel-btn" v-close-popup="true" class="q-mb-md text-bold"
                :label="t('alerts.cancel')" text-color="light-text" padding="sm md" no-caps
                @click="$emit('cancel:hideform')" />
              <q-btn data-test="add-alert-submit-btn" :label="t('alerts.save')"
                class="q-mb-md text-bold no-border q-ml-md" color="secondary" padding="sm xl" type="submit" no-caps />
            </div>
          </q-form>
        </div>
        <div style="width: 400px; height: 200px; position: sticky; top: 0" class="q-mb-lg q-px-md">
          <div class="text-bold q-pb-xs text-grey-9">Preview</div>
          <preview-alert style="border: 1px solid #ececec" ref="previewAlertRef" :formData="formData"
            :query="previewQuery" :aggregationEnabled="isAggregationEnabled" />
        </div>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import {
  defineComponent,
  ref,
  onMounted,
  watch,
  type Ref,
  computed,
  nextTick,
} from "vue";

import "monaco-editor/esm/vs/editor/editor.all.js";
import "monaco-editor/esm/vs/basic-languages/sql/sql.contribution.js";
import "monaco-editor/esm/vs/basic-languages/sql/sql.js";

import alertsService from "../../services/alerts";
import { useI18n } from "vue-i18n";
import { useStore } from "vuex";
import { useQuasar, debounce } from "quasar";
import streamService from "../../services/stream";
import { Parser } from "node-sql-parser/build/mysql";
import segment from "../../services/segment_analytics";
import ScheduledAlert from "./ScheduledAlert.vue";
import RealTimeAlert from "./RealTimeAlert.vue";
import VariablesInput from "./VariablesInput.vue";
import { getUUID } from "@/utils/zincutils";
import { cloneDeep } from "lodash-es";
import { useRouter } from "vue-router";
import PreviewAlert from "./PreviewAlert.vue";

const defaultValue: any = () => {
  return {
    name: "",
    stream_type: "",
    stream_name: "",
    is_real_time: "false",
    query_condition: {
      conditions: [
        {
          column: "",
          operator: "=",
          value: "",
          type: "",
          id: getUUID(),
        },
      ],
      sql: "",
      promql: null,
      type: "custom",
      aggregation: {
        group_by: [""],
        function: "avg",
        having: {
          column: "",
          operator: ">=",
          value: 0,
        },
      },
    },
    trigger_condition: {
      period: 10,
      operator: ">=",
      threshold: 3,
      silence: 10,
    },
    destinations: [],
    context_attributes: {},
    enabled: true,
    description: "",
  };
};
let callAlert: Promise<{ data: any }>;
export default defineComponent({
  name: "ComponentAddUpdateAlert",
  props: {
    modelValue: {
      type: Object,
      default: () => defaultValue(),
    },
    isUpdated: {
      type: Boolean,
      default: false,
    },
    destinations: {
      type: Array,
      default: () => [],
    },
  },
  emits: ["update:list", "cancel:hideform"],
  components: {
    ScheduledAlert,
    RealTimeAlert,
    VariablesInput,
    PreviewAlert,
  },
  setup(props) {
    const store: any = useStore();
    let beingUpdated: boolean = false;
    const addAlertForm: any = ref(null);
    const disableColor: any = ref("");
    const formData: any = ref(defaultValue());
    const indexOptions = ref([]);
    const schemaList = ref([]);
    const streams: any = ref({});
    const { t } = useI18n();
    const q = useQuasar();
    const editorRef: any = ref(null);
    const filteredColumns: any = ref([]);
    const filteredStreams: Ref<string[]> = ref([]);
    let editorobj: any = null;
    var sqlAST: any = ref(null);
    const selectedRelativeValue = ref("1");
    const selectedRelativePeriod = ref("Minutes");
    const relativePeriods: any = ref(["Minutes"]);
    var triggerCols: any = ref([]);
    const selectedDestinations = ref("slack");
    const originalStreamFields: any = ref([]);
    const isAggregationEnabled = ref(false);
    var triggerOperators: any = ref([
      "=",
      "!=",
      ">=",
      "<=",
      ">",
      "<",
      "Contains",
      "NotContains",
    ]);
    const isFetchingStreams = ref(false);
    const streamTypes = ["logs", "metrics", "traces"];
    const editorUpdate = (e: any) => {
      formData.value.sql = e.target.value;
    };

    const previewQuery = ref("");

    const addAlertFormRef = ref(null);

    const router = useRouter();
    const scheduledAlertRef: any = ref(null);

    const plotChart: any = ref(null);

    const previewAlertRef: any = ref(null);

    const streamFieldsMap = computed(() => {
      const map: any = {};
      originalStreamFields.value.forEach((field: any) => {
        map[field.value] = field;
      });
      return map;
    });

    const showPreview = computed(() => {
      return (
        formData.value.name &&
        formData.value.stream_type &&
        formData.value.stream_name &&
        areConditionsValid
      );
    });

    const areConditionsValid = computed(() => {
      return formData.value.query_condition.conditions.every(
        (condition: any) => {
          condition.column.trim().length &&
            condition.operator.trim().length &&
            condition.value.trim().length;
        }
      );
    });

    const updateCondtions = (e: any) => {
      try {
        const ast = parser.astify(e.target.value);
        if (ast) sqlAST.value = ast;
        else return;

        // If sqlAST.value.columns is not type of array then return
        if (!sqlAST.value) return;
        if (!Array.isArray(sqlAST.value?.columns)) return;

        sqlAST.value.columns.forEach(function (item: any) {
          let val;
          if (item["as"] === undefined || item["as"] === null) {
            val = item["expr"]["column"];
          } else {
            val = item["as"];
          }
          if (!triggerCols.value.includes(val)) {
            triggerCols.value.push(val);
          }
        });
      } catch (e) {
        console.log("Alerts: Error while parsing SQL query");
      }
    };
    const editorData = ref("");
    const prefixCode = ref("");
    const suffixCode = ref("");
    let parser = new Parser();

    onMounted(async () => { });

    const updateEditorContent = (stream_name: string) => {
      triggerCols.value = [];
      if (!stream_name) return;

      if (editorData.value) {
        editorData.value = editorData.value
          .replace(prefixCode.value, "")
          .trim();
        editorData.value = editorData.value
          .replace(suffixCode.value, "")
          .trim();
      }

      if (!props.isUpdated) {
        prefixCode.value = `select * from`;
        suffixCode.value = "'" + formData.value.stream_name + "'";
        const someCode = `${prefixCode.value} ${editorData.value} ${suffixCode.value}`;
      }

      const selected_stream: any = schemaList.value.filter(
        (stream) => stream["name"] === stream_name
      );
      selected_stream[0].schema.forEach(function (item: any, index: any) {
        triggerCols.value.push(item.name);
      });
    };

    const updateStreamFields = (stream_name: any) => {
      let streamCols: any = [];
      const column: any = schemaList.value.find(
        (schema: any) => schema.name === stream_name
      );

      if (column && Array.isArray(column?.schema)) {
        streamCols = column.schema.map((column: any) => ({
          label: column.name,
          value: column.name,
          type: column.type,
        }));
      }

      originalStreamFields.value = [...streamCols];
      filteredColumns.value = [...streamCols];

      onInputUpdate("stream_name", stream_name);
    };

    watch(
      triggerCols.value,
      () => {
        filteredColumns.value = [...triggerCols.value];
      },
      { immediate: true }
    );
    const filterColumns = (options: any[], val: String, update: Function) => {
      let filteredOptions: any[] = [];
      if (val === "") {
        update(() => {
          filteredOptions = [...options];
        });
        return filteredOptions;
      }
      update(() => {
        const value = val.toLowerCase();
        filteredOptions = options.filter(
          (column: any) => column.toLowerCase().indexOf(value) > -1
        );
      });
      return filteredOptions;
    };
    const updateStreams = (resetStream = true) => {
      if (formData.value.stream_type === "metrics") {
        isAggregationEnabled.value = true;
      }
      if (resetStream) formData.value.stream_name = "";
      if (streams.value[formData.value.stream_type]) {
        schemaList.value = streams.value[formData.value.stream_type];
        indexOptions.value = streams.value[formData.value.stream_type].map(
          (data: any) => {
            return data.name;
          }
        );
        return;
      }

      if (!formData.value.stream_type) return Promise.resolve();

      isFetchingStreams.value = true;
      return streamService
        .nameList(
          store.state.selectedOrganization.identifier,
          formData.value.stream_type,
          true
        )
        .then((res) => {
          streams.value[formData.value.stream_type] = res.data.list;
          schemaList.value = res.data.list;
          indexOptions.value = res.data.list.map((data: any) => {
            return data.name;
          });

          if (formData.value.stream_name)
            updateStreamFields(formData.value.stream_name);
          return Promise.resolve();
        })
        .catch(() => Promise.reject())
        .finally(() => (isFetchingStreams.value = false));
    };

    const filterStreams = (val: string, update: any) => {
      filteredStreams.value = filterColumns(indexOptions.value, val, update);
    };

    const addField = () => {
      formData.value.query_condition.conditions.push({
        column: "",
        operator: "=",
        value: "",
        id: getUUID(),
      });
    };

    const removeField = (field: any) => {
      if (formData.value.query_condition.conditions.length === 1) return;

      formData.value.query_condition.conditions =
        formData.value.query_condition.conditions.filter(
          (_field: any) => _field.id !== field.id
        );
    };

    const addVariable = () => {
      formData.value.context_attributes.push({
        name: "",
        value: "",
        id: getUUID(),
      });
    };

    const removeVariable = (variable: any) => {
      formData.value.context_attributes =
        formData.value.context_attributes.filter(
          (_variable: any) => _variable.id !== variable.id
        );
    };

    const previewAlert = async () => {
      previewQuery.value = generateSqlQuery();
      await nextTick();
      previewAlertRef.value.refreshData();
    };

    const generateSqlQuery = () => {
      // SELECT histgoram(_timestamp, '1 minute') AS zo_sql_key, COUNT(*) as zo_sql_val FROM _rundata WHERE geo_info_country='india' GROUP BY zo_sql_key ORDER BY zo_sql_key ASC

      // SELECT histgoram(_timestamp, '1 minute') AS zo_sql_key, avg(action_error_count) as zo_sql_val, geo_info_city FROM _rundata WHERE geo_info_country='india' GROUP BY zo_sql_key,geo_info_city ORDER BY zo_sql_key ASC;
      let query = "SELECT histogram(_timestamp) AS zo_sql_key,";

      let whereClause = formData.value.query_condition.conditions
        .map((condition: any) => {
          if (condition.column && condition.operator && condition.value) {
            console.log(condition.column, condition.value);
            const value =
              streamFieldsMap.value[condition.column].type === "Int64"
                ? condition.value
                : `'${condition.value}'`;
            return `${condition.column}${condition.operator}${value}`;
          }
        })
        .join(" AND ");

      whereClause = whereClause?.trim().length ? " WHERE " + whereClause : "";

      if (!isAggregationEnabled.value) {
        query +=
          ` COUNT(*) as zo_sql_val FROM ${formData.value.stream_name} ` +
          whereClause +
          " GROUP BY zo_sql_key ORDER BY zo_sql_key ASC";
      } else {
        const aggFn = formData.value.query_condition.aggregation.function;
        const column = formData.value.query_condition.aggregation.having.column;
        let groupBy = "";
        formData.value.query_condition.aggregation.group_by.forEach(
          (column: any) => {
            if (column.trim().length) groupBy += `,${column}`;
          }
        );

        console.log("groupBy", groupBy, groupBy?.trim().length);

        query +=
          ` ${aggFn}(${column}) as zo_sql_val ${groupBy} FROM ${formData.value.stream_name} ` +
          whereClause +
          ` GROUP BY zo_sql_key ${groupBy} ORDER BY zo_sql_key ASC`;
      }

      return query;
    };

    const debouncedPreviewAlert = debounce(previewAlert, 500);

    const onInputUpdate = async (name: string, value: any) => {
      console.log(name, cloneDeep(formData.value.query_condition.conditions));
      if (showPreview.value && validateInputs(getAlertPayload(), false)) {
        debouncedPreviewAlert();
      }
    };

    const getAlertPayload = () => {
      const payload = cloneDeep(formData.value);

      payload.is_real_time = payload.is_real_time === "true";

      payload.context_attributes = {};

      payload.query_condition.type = payload.is_real_time
        ? "custom"
        : formData.value.query_condition.type;

      formData.value.context_attributes.forEach((attr: any) => {
        if (attr.key?.trim() && attr.value?.trim())
          payload.context_attributes[attr.key] = attr.value;
      });

      payload.trigger_condition.threshold = parseInt(
        formData.value.trigger_condition.threshold
      );

      payload.trigger_condition.period = parseInt(
        formData.value.trigger_condition.period
      );

      payload.trigger_condition.silence = parseInt(
        formData.value.trigger_condition.silence
      );

      payload.description = formData.value.description.trim();

      if (!isAggregationEnabled.value) {
        payload.query_condition.aggregation = null;
      }

      if (scheduledAlertRef.value.tab === "sql") {
        payload.query_condition.conditions = [];
      }

      return payload;
    };

    const validateInputs = (input: any, notify: boolean = true) => {
      if (
        Number(input.trigger_condition.silence) < 1 ||
        isNaN(Number(input.trigger_condition.silence))
      ) {
        notify &&
          q.notify({
            type: "negative",
            message: "Silence Notification should be greater than 0",
            timeout: 1500,
          });
        return false;
      }

      if (input.is_real_time) return true;

      if (
        Number(input.trigger_condition.period) < 1 ||
        isNaN(Number(input.trigger_condition.period))
      ) {
        notify &&
          q.notify({
            type: "negative",
            message: "Period should be greater than 0",
            timeout: 1500,
          });
        return false;
      }

      if (input.query_condition.aggregation) {
        if (
          !input.query_condition.aggregation.having.value.toString().trim()
            .length ||
          !input.query_condition.aggregation.having.column ||
          !input.query_condition.aggregation.having.operator
        ) {
          notify &&
            q.notify({
              type: "negative",
              message: "Threshold should not be empty",
              timeout: 1500,
            });
          return false;
        }

        return true;
      }

      if (
        !input.trigger_condition.threshold.toString().trim().length ||
        !input.trigger_condition.operator
      ) {
        notify &&
          q.notify({
            type: "negative",
            message: "Threshold should not be empty",
            timeout: 1500,
          });
        return false;
      }

      return true;
    };

    return {
      t,
      q,
      disableColor,
      beingUpdated,
      formData,
      addAlertForm,
      store,
      indexOptions,
      editorRef,
      editorobj,
      prefixCode,
      suffixCode,
      editorData,
      selectedRelativeValue,
      selectedRelativePeriod,
      relativePeriods,
      editorUpdate,
      updateCondtions,
      updateStreamFields,
      updateEditorContent,
      triggerCols,
      triggerOperators,
      sqlAST,
      schemaList,
      filteredColumns,
      streamTypes,
      streams,
      updateStreams,
      isFetchingStreams,
      filteredStreams,
      filterStreams,
      addField,
      removeField,
      removeVariable,
      addVariable,
      selectedDestinations,
      scheduledAlertRef,
      router,
      isAggregationEnabled,
      plotChart,
      previewAlert,
      addAlertFormRef,
      validateInputs,
      getAlertPayload,
      onInputUpdate,
      showPreview,
      streamFieldsMap,
      previewQuery,
      previewAlertRef,
    };
  },

  created() {
    this.formData.ingest = ref(false);
    this.formData = { ...defaultValue, ...this.modelValue };
    this.formData.is_real_time = this.formData.is_real_time.toString();
    this.beingUpdated = this.isUpdated;
    this.updateStreams(false)?.then(() => {
      this.updateEditorContent(this.formData.stream_name);
    });
    if (
      this.modelValue &&
      this.modelValue.name != undefined &&
      this.modelValue.name != ""
    ) {
      this.beingUpdated = true;
      this.disableColor = "grey-5";
      this.formData = this.modelValue;
      this.isAggregationEnabled = !!this.formData.query_condition.aggregation;
    }

    this.formData.is_real_time = this.formData.is_real_time.toString();
    this.formData.context_attributes = Object.keys(
      this.formData.context_attributes
    ).map((attr) => {
      return {
        key: attr,
        value: this.formData.context_attributes[attr],
        id: getUUID(),
      };
    });
    this.formData.query_condition.conditions =
      this.formData.query_condition.conditions.map((condition: any) => {
        return {
          ...condition,
          id: getUUID(),
        };
      });
  },

  computed: {
    getFormattedDestinations: function () {
      return this.destinations.map((destination: any) => {
        return destination.name;
      });
    },
  },
  methods: {
    onRejected(rejectedEntries: string | any[]) {
      this.q.notify({
        type: "negative",
        message: `${rejectedEntries.length} file(s) did not pass validation constraints`,
      });
    },
    onSubmit() {
      if (this.formData.stream_name == "") {
        this.q.notify({
          type: "negative",
          message: "Please select stream name.",
          timeout: 1500,
        });
        return false;
      }

      this.addAlertForm.validate().then((valid: any) => {
        if (!valid) {
          return false;
        }

        const payload = this.getAlertPayload();
        if (!this.validateInputs(payload)) return;

        const dismiss = this.q.notify({
          spinner: true,
          message: "Please wait...",
          timeout: 2000,
        });

        if (this.beingUpdated) {
          callAlert = alertsService.update(
            this.store.state.selectedOrganization.identifier,
            payload.stream_name,
            payload.stream_type,
            payload
          );
          callAlert
            .then((res: { data: any }) => {
              this.formData = { ...defaultValue };
              this.$emit("update:list");
              this.addAlertForm.resetValidation();
              dismiss();
              this.q.notify({
                type: "positive",
                message: `Alert updated successfully.`,
              });
            })
            .catch((err: any) => {
              dismiss();
              this.q.notify({
                type: "negative",
                message: err.response?.data?.error || err.response?.data?.message,
              });
            });
          segment.track("Button Click", {
            button: "Update Alert",
            user_org: this.store.state.selectedOrganization.identifier,
            user_id: this.store.state.userInfo.email,
            stream_name: this.formData.stream_name,
            alert_name: this.formData.name,
            page: "Add/Update Alert",
          });
          return;
        } else {
          callAlert = alertsService.create(
            this.store.state.selectedOrganization.identifier,
            payload.stream_name,
            payload.stream_type,
            payload
          );

          callAlert
            .then((res: { data: any }) => {
              this.formData = { ...defaultValue };
              this.$emit("update:list");
              this.addAlertForm.resetValidation();
              dismiss();
              this.q.notify({
                type: "positive",
                message: `Alert saved successfully.`,
              });
            })
            .catch((err: any) => {
              dismiss();
              this.q.notify({
                type: "negative",
                message: err.response?.data?.error || err.response?.data?.message,
              });
            });
          segment.track("Button Click", {
            button: "Save Alert",
            user_org: this.store.state.selectedOrganization.identifier,
            user_id: this.store.state.userInfo.email,
            stream_name: this.formData.stream_name,
            alert_name: this.formData.name,
            page: "Add/Update Alert",
          });
        }

      });
    },
  },
});
</script>

<style scoped lang="scss">
#editor {
  width: 100%;
  min-height: 5rem;
  // padding-bottom: 14px;
  resize: both;
}

.alert-condition {

  .__column,
  .__value {
    width: 250px;
  }

  .__operator {
    width: 100px;
  }
}
</style>
<style lang="scss">
.no-case .q-field__native span {
  text-transform: none !important;
}

.no-case .q-field__input {
  text-transform: none !important;
}

.add-alert-form {
  .q-field--dense .q-field__control {

    .q-field__native span {
      overflow: hidden;
    }
  }

  .alert-condition .__column .q-field__control .q-field__native span {
    max-width: 152px;
    text-overflow: ellipsis;
    text-align: left;
    white-space: nowrap;
  }

  .q-field__bottom {
    padding: 2px 0;
  }
}

.silence-notification-input,
.threshould-input {
  .q-field--filled .q-field__control {
    background-color: transparent !important;
  }

  .q-field--dark .q-field__control {
    background-color: rgba(255, 255, 255, 0.07) !important;
  }
}
</style>
