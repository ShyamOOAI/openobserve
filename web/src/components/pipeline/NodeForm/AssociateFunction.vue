<!-- Copyright 2023 OpenObserve Inc.

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
  <div
    data-test="add-stream-routing-section"
    class="full-width full-height"
    :class="store.state.theme === 'dark' ? 'bg-dark' : 'bg-white'"
  >
    <div class="stream-routing-title q-pb-sm q-pl-md">
      {{ t("pipeline.associateFunction") }}
    </div>
    <q-separator />
    <div class="q-px-md">
      <q-select
          color="input-border"
          class="q-py-sm showLabelOnTop no-case tw-w-full "
          stack-label
          outlined
          filled
          dense
          v-model="selected"
          :options="filteredOptions"
          use-input
          input-debounce="300"
          @filter="filterOptions"


          label="Select Previous Node"
          clearable
        >
        <template v-slot:option="scope">
  <q-item
    v-bind="scope.itemProps"
    v-if="!scope.opt.isGroup"
    class="full-width"
    :style="{ backgroundColor: scope.opt.color  }"
    style="color: black;"
  >              
    <q-item-section avatar class="w-full">
      <q-img
        :src="scope.opt.icon"
        style="width: 24px; height: 24px"
      />
    </q-item-section>
    
    <div class="flex tw-justify-between tw-w-full"  >
      <q-item-section>
        <q-item-label v-html="scope.opt.label"></q-item-label>
      </q-item-section>
      <q-item-section>
        <q-item-label class="tw-ml-auto" v-html="scope.opt.node_type"></q-item-label>
      </q-item-section>
    </div>
  </q-item>

  <!-- Render non-selectable group headers -->
  <q-item v-else   :class="store.state.theme === 'dark' ? 'bg-dark' : 'bg-white'">
    <q-item-section >
      <q-item-label v-html="scope.opt.label" />
    </q-item-section>
  </q-item>
</template>

        </q-select>
      </div>
    <div v-if="loading">
      <q-spinner
        v-if="loading"
        color="primary"
        size="40px"
        style="
          position: absolute;
          top: 50%;
          left: 50%;
          transform: translate(-50%, -50%);
        "
      />
    </div>
    <div v-else class="stream-routing-container full-width q-pa-md">
      <q-toggle
        data-test="create-function-toggle"
        class="q-mb-sm"
        :label="isUpdating ? 'Edit function' : 'Create new function'"
        v-model="createNewFunction"
      />
      <q-form @submit="saveFunction">
        <div
          v-if="!createNewFunction"
          class="flex justify-start items-center full-width"
          style="padding-top: 0px"
        >
          <div
            data-test="associate-function-select-function-input"
            class="alert-stream-type o2-input q-mr-sm full-width"
            style="padding-top: 0"
          >
            <q-select
              v-model="selectedFunction"
              :options="filteredFunctions"
              :label="t('function.selectFunction') + ' *'"
              :popup-content-style="{ textTransform: 'lowercase' }"
              color="input-border"
              bg-color="input-bg"
              class="q-py-sm showLabelOnTop no-case"
              stack-label
              outlined
              filled
              dense
              use-input
              input-debounce="300"
              :rules="[(val: any) => !!val || 'Field is required!']"
              style="min-width: 220px"
              v-bind:readonly="isUpdating"
              v-bind:disable="isUpdating"
              :error-message="
                selectedFunction ? 'Function is already associated' : ''
              "
              @filter="filterFunctions"
              :error="functionExists"
            />
    
          </div>
        </div>

        <div v-if="createNewFunction" class="pipeline-add-function">
          <AddFunction
            ref="addFunctionRef"
            :is-updated="isUpdating"
            @update:list="onFunctionCreation"
            @cancel:hideform="cancelFunctionCreation"
          />
        </div>

        <div class="o2-input full-width" style="padding-top: 12px" v-if="!createNewFunction">
          <q-toggle
            data-test="pipeline-function-after-flattening-toggle"
            class="q-mb-sm"
            :label="t('pipeline.flatteningLbl')"
            v-model="afterFlattening"
          />
        </div>
        <div v-else class="q-pb-sm container text-body2" style="width: 500px;">
          {{t("alerts.newFunctionAssociationMsg")}}
        </div>

        <div
          class="flex justify-start full-width"
          :class="store.state.theme === 'dark' ? 'bg-dark' : 'bg-white'"
        >
          <q-btn
            data-test="associate-function-cancel-btn"
            class="text-bold"
            :label="t('alerts.cancel')"
            text-color="light-text"
            padding="sm md"
            no-caps
            @click="openCancelDialog"
          />
          <q-btn
            data-test="associate-function-save-btn"
            :label="createNewFunction ? t('alerts.createFunction') : t('alerts.save')"
            class="text-bold no-border q-ml-md"
            color="secondary"
            padding="sm xl"
            no-caps
            type="submit"
          />
          <q-btn
          v-if="pipelineObj.isEditNode"
          data-test="associate-function-delete-btn"
            :label="t('pipeline.deleteNode')"
            class="text-bold no-border q-ml-md"
            color="negative"
            padding="sm xl"
            no-caps
            @click="openDeleteDialog"
          />
        </div>
      </q-form>
    </div>
  </div>
  <confirm-dialog
    v-model="dialog.show"
    :title="dialog.title"
    :message="dialog.message"
    @update:ok="dialog.okCallback"
    @update:cancel="dialog.show = false"
  />
</template>
<script lang="ts" setup>
import {
  ref,
  type Ref,
  defineEmits,
  onBeforeMount,
  watch,
  nextTick,
  defineAsyncComponent,
  onMounted,
  computed,
} from "vue";
import { useI18n } from "vue-i18n";
import { useStore } from "vuex";
import ConfirmDialog from "@/components/ConfirmDialog.vue";
import useDragAndDrop from "@/plugins/pipelines/useDnD";
import { useQuasar } from "quasar";
import { getImageURL } from "@/utils/zincutils";





interface RouteCondition {
  column: string;
  operator: string;
  value: any;
  id: string;
}

interface StreamRoute {
  sourceStreamName: string;
  destinationStreamName: string;
  sourceStreamType: string;
  destinationStreamType: string;
  conditions: RouteCondition[];
}


const AddFunction = defineAsyncComponent(
  () => import("@/components/functions/AddFunction.vue"),
);

const props = defineProps({
  functions: {
    type: Array,
    required: true,
    default: () => {
      return [];
    },
  },
  associatedFunctions: {
    type: Array,
    required: true,
    default: () => {
      return [];
    },
  },
});

const emit = defineEmits([
  "update:node",
  "cancel:hideform",
  "delete:node",
  "add:function",
]);

const { t } = useI18n();

const { addNode, pipelineObj , deletePipelineNode, formattedOptions,   filteredOptions, filterOptions,getParentNode ,currentSelectedParentNode} = useDragAndDrop();

const addFunctionRef: any = ref(null);

const isUpdating = ref(false);

const selectedFunction = ref((pipelineObj.currentSelectedNodeData?.data as { name?: string })?.name || "");

const loading = ref(false);

const afterFlattening = ref((pipelineObj.currentSelectedNodeData?.data as { after_flatten?: boolean })?.after_flatten || false);

const filteredFunctions: Ref<any[]> = ref([]);

const createNewFunction = ref(false);
const q = useQuasar();

const store = useStore();

const functionExists = ref(false);
const selected = ref(null);

watch(selected, (newValue:any) => {
      pipelineObj.userSelectedNode = newValue; 
});

const nodeLink = ref({
  from: "",
  to: "",
});

const dialog = ref({
  show: false,
  title: "",
  message: "",
  okCallback: () => {},
});

watch(
      () => props.functions,
      (newVal) => {
        filteredFunctions.value = [...newVal].sort((a:any, b:any) => {
          return a.localeCompare(b);
        });
      },
      {
        deep: true,
        immediate: true,
      }
    );
onMounted(()=>{
  if(pipelineObj.isEditNode){
    const selectedParentNode = currentSelectedParentNode();
    if(selectedParentNode){
      selected.value = selectedParentNode;

    }
  }
  else{
    pipelineObj.userSelectedNode = {};
    selected.value = null;
  }
})


const openCancelDialog = () => {
  if(!isUpdating){
    if(createNewFunction.value == true  && addFunctionRef.value.formData.name == "" && addFunctionRef.value.formData.function == "") {
    createNewFunction.value = false;
    return;
  }
  }


  dialog.value.show = true;
  dialog.value.title = "Discard Changes";
  dialog.value.message = "Are you sure you want to cancel changes?";
  dialog.value.okCallback = () => emit("cancel:hideform");
};

const openDeleteDialog = () => {
  dialog.value.show = true;
  dialog.value.title = "Delete Node";
  dialog.value.message =
    "Are you sure you want to delete function association?";
  dialog.value.okCallback = deleteFunction;
};

const saveFunction = () => {
  functionExists.value = false;
  
  if (createNewFunction.value) {
    if(addFunctionRef.value.formData.name == "" ){
      return;
    }
    if(addFunctionRef.value.formData.function == ""){
     q.notify({
        message: "Function is required",
        color: "negative",
        position: "bottom",
        timeout: 2000,
      });
      return;
    }
    addFunctionRef.value.onSubmit();
    return;
  }

  if (
    !isUpdating.value &&
    selectedFunction.value &&
    props.associatedFunctions.includes(selectedFunction.value)
  ) {
    functionExists.value = true;
    return;
  }

  const functionNode = {
    name: selectedFunction.value,
    after_flatten: afterFlattening.value,
  };
  addNode(functionNode);
  // emit("update:node", {
  //   data: { name: selectedFunction.value, order: functionOrder.value },
  //   link: nodeLink.value,
  // });
  emit("cancel:hideform");
};

const onFunctionCreation = async (_function: any) => {
  // Assign newly created function to the block
  createNewFunction.value = false;
  emit("add:function", _function);
  await nextTick();
  selectedFunction.value = _function.name;
};

const cancelFunctionCreation = () => {
  emit("cancel:hideform");
};

const saveUpdatedLink = (link: { from: string; to: string }) => {
  nodeLink.value = link;
};

const deleteFunction = () => {
  deletePipelineNode (pipelineObj.currentSelectedNodeID);

  emit("cancel:hideform");
};
const filterFunctions = (val:any, update:any) => {
      const filtered = props.functions
        .filter((func:any) =>
          func.toLowerCase().includes(val.toLowerCase())
        )
        .sort((a:any, b:any) => a.localeCompare(b));



      update(() => {
        filteredFunctions.value = filtered;
      });
    };
</script>

<style scoped>
.stream-routing-title {
  font-size: 20px;
  padding-top: 16px;
}
.stream-routing-container {
  border-radius: 8px;
  /* box-shadow: 0px 0px 10px 0px #d2d1d1; */
}
</style>

<style lang="scss">
.pipeline-add-function {
  .add-function-header,
  .q-separator {
    display: none;
  }

  .add-function-actions {
    display: none;
  }

  .add-function-name-input {
    width: 100%;
    margin-left: 0px !important;

    label {
      width: 100%;
      padding-left: 0;
    }
  }
}
</style>
