<template>
  <div>
    <div class="vss-list">
      <v-search
        class="vss-list-search"
        v-if="search"
        v-model="searchL"
      ></v-search>
      <v-list
        :enable-counter="false"
        :has-children="false"
        :items="filteredListL"
        @updated-item="updateItem"
      ></v-list>
      <div class="vss-footer">
        <div v-if="toggleAll">
          <v-selectAll
            :items="listLeft"
            @update-select-all="updateLeftSelectAll"
          ></v-selectAll>
        </div>
      </div>
    </div>
    <v-separator></v-separator>
    <div class="vss-list">
      <v-search
        class="vss-list-search"
        v-if="search"
        v-model="searchR"
      ></v-search>
      <v-list
        :enable-counter="false"
        :has-children="false"
        :items="filteredListR"
        @updated-item="updateItem"
      ></v-list>
      <div class="vss-footer">
        <div v-if="toggleAll">
          <v-deselectAll
            :items="listRight"
            @update-deselect-all="updateRightDeselectAll"
          ></v-deselectAll>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { normalizeText, clone, reorder, removeItemArray } from "../utils";

const vSelectAll = require("../components/selectAll.vue").default;
const vDeselectAll = require("../components/deselectAll.vue").default;
const vSearch = require("../components/search.vue").default;
const vList = require("../components/list.vue").default;
const vSeparator = require("../components/separator.vue").default;
const mixin = require("../mixin").default;

export default {
  name: "mirror-select-sides",
  display: "Mirror select sides",
  mixins: [mixin],
  components: {
    vSelectAll,
    vDeselectAll,
    vSearch,
    vSeparator,
    vList
  },
  props: {
    list: {
      required: true,
      type: [Array, Object]
    },
    search: {
      type: Boolean
    },
    toggleAll: {
      type: Boolean
    },
    selected: {
      type: Array
    },
    selectedItem: {
      type: Array
    },
    selectedParent: {
      type: Array
    },
    orderBy: {
      type: String
    },
    sortSelectedUp: {
      type: Boolean
    }
  },
  methods: {
    updateLeftSelectAll() {
      let vm = this;

      vm.listLeft.map(item => {
        if (item.visible === true) {
          vm.updateItem(item, {}, true);
        }
      });
    },
    updateRightDeselectAll() {
      let vm = this;

      vm.listRight.map(item => {
        if (item.visible === true) {
          vm.updateItem(item, {}, false);
        }
      });
    },
    updateItem(item, parent, selected) {
      let dataSelected = clone(this.dataSelected);

      if (Object.keys(parent).length > 0) {
        if (dataSelected[parent.value] === undefined) {
          if (parent.visible) {
            dataSelected[parent.value] = [];
          }
        }

        if (selected) {
          if (item.visible) {
            dataSelected[parent.value].push(item.value);
          }
        } else {
          if (item.visible) {
            dataSelected[parent.value] = removeItemArray(
              dataSelected[parent.value],
              item.value
            );
          }
        }
      } else {
        if (selected) {
          if (dataSelected[item.value] === undefined) {
            dataSelected[item.value] = [];
          }
        } else {
          delete dataSelected[item.value];
        }
      }

      this.$set(this, "dataSelected", dataSelected);
    },
    getSelectedItem() {
      let vm = this;
      let propSelected = [];
      let concat = [];
      let selected = this.selected !== undefined ? vm.selected : [];
      let selectedItem =
        this.selectedItem !== undefined ? this.selectedItem : [];

      selected.forEach(parent => {
        propSelected.push(parent);
      });

      console.log(selected);

      concat = [...propSelected, ...selectedItem];

      return [...new Set(concat)];
    }
  },
  beforeMount() {
    this.$set(this, "dataList", this.list);

    // Organiza os "defaultSelected" para o reorder
    let selectedsItem = this.getSelectedItem();

    let dataList = this.dataList.filter(item => {
      let value = item.value;

      if (selectedsItem.indexOf(value) >= 0) {
        item.selectedDefault = true;
      } else {
        item.selectedDefault = false;
      }

      this.dataList = reorder(this, this.dataList);

      return item;
    });

    // Organiza o array de selecionados
    dataList.map(item => {
      if (item.selectedDefault) {
        if (this.dataSelected[item.value] === undefined) {
          this.dataSelected[item.value] = [];
        }
      }

      if (item.children) {
        item.children.map(children => {
          if (children.selectedDefault) {
            this.dataSelected[item.value].push(children.value);
          }
        });
      }
    });

    this.$set(this, "dataList", this.dataList);
  },
  mounted() {
    // Organiza a listLeft
    this.listLeft = this.dataList.filter(item => {
      item.visible = true;
      return item;
    });
  },
  computed: {
    filteredListL() {
      // let vm = this;
      let search = normalizeText(this.searchL);
      let selected = Object.keys(this.dataSelected);

      let listLeft = clone(this.listLeft);

      listLeft = listLeft.filter(item => {
        let label = normalizeText(item.label);

        // Has selected
        if (selected.indexOf(item.value) >= 0) {
          item.selected = true;
          item.visible = false;
        } else {
          item.selected = false;
          item.visible = true;
        }

        // Has search
        if (label.includes(search) && item.visible === true) {
          item.visible = true;
        } else {
          item.visible = false;
        }

        return item;
      });

      this.$set(this, "listLeft", listLeft);

      return listLeft;
    },
    filteredListR() {
      let vm = this;
      let search = normalizeText(vm.searchR);
      let selected = Object.keys(this.dataSelected);
      let listRight = clone(vm.listLeft);

      listRight = listRight.filter(item => {
        let label = normalizeText(item.label);

        // Has selected
        if (selected.indexOf(item.value) >= 0) {
          item.selected = true;
          item.visible = true;
        } else {
          item.selected = false;
          item.visible = false;
        }

        // Has search
        if (label.includes(search) && item.visible === true) {
          item.visible = true;
        } else {
          item.visible = false;
        }

        return item;
      });

      this.$set(this, "listRight", listRight);

      return listRight;
    }
  },
  data() {
    return {
      dataList: [],
      dataSelected: {},
      listLeft: [],
      listRight: [],
      searchL: "",
      searchR: ""
    };
  }
};
</script>