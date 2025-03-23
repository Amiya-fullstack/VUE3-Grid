/* eslint-disable */
<template>
  <div>
    <!-- Super Header -->
    <div class="super-header">
      <!-- Global Search Bar -->
      <div v-if="tableConfig.tableFilter" class="global-search">
        <input type="text" v-model="globalSearch" @input="applyGlobalFilter" placeholder="Search..." />
      </div>

      <!-- Clear All Filters Button -->
      <div class="clear-filters">
        <button @click="clearAllFilters">
          <v-icon class="mdi mdi-filter-remove-outline"></v-icon> Clear All Filters
        </button>
      </div>

      <!-- Edit Button -->
      <div class="edit-button" v-if="tableConfig.tableEdit">
        <button @click="toggleEditOptions">
          <v-icon v-if="isEditing" class="mdi mdi-cancel"></v-icon>
          <v-icon v-else class="mdi mdi-pencil"></v-icon>
        </button>
        <button v-if="isEditing" @click="saveEdits">
          <v-icon class="mdi mdi-content-save"></v-icon>
        </button>
      </div>
    </div>

    <div class="table">
      <!-- Table Header -->
      <div class="table-header">
        <div class="header-cell">Expand</div>
        <div v-if="tableConfig.rowSelection" class="header-cell">
          <input type="checkbox" @change="handleSelectAll" v-if="tableConfig.selectionAll" />
        </div>
        <div
          class="header-cell"
          v-for="column in columnConfig"
          :key="column.key"
          @click="column.sortable && handleSort(column.key)"
        >
          {{ column.label }}
          <span v-if="column.sortable"> {{ getSortIcon(column.key) }} </span>
          <span v-if="column.filterable" @click.stop="toggleFilterOptions(column.key)">
            <v-icon class="mdi mdi-filter"></v-icon>
          </span>
          <div v-if="showFilterOptions[column.key]" class="filter-options">
            <input type="text" v-model="filterInputValues[column.key]" placeholder="Filter..." />
            <button @click="applyFilter(column.key)">Apply Filter</button>
          </div>
        </div>
      </div>

      <!-- Table Body (Recursive Rows) -->
      <div
        class="table-row-wrapper"
        v-for="(row, index) in sortedFilteredData"
        :key="row.id"
        :class="getRowClass(index)"
      >
        <CustomTableRow
          :row="row"
          :column-config="columnConfig"
          :table-config="tableConfig"
          :expanded-rows="expandedRows"
          :selected-rows="selectedRows"
          :is-editing="isEditing"
          :row-index="index"
          ref="customTableRow"
          @updateRow="handleRowUpdate"
          @selectionChanged="handleSelectionChange"
          @toggleChildren="toggleChildren"
        />
      </div>
    </div>
  </div>
</template>

<script>
import { reactive, toRefs } from 'vue';
import CustomTableRow from "./CustomTableRow.vue";
import '@mdi/font/css/materialdesignicons.css';

export default {
  name: "CustomTable",
  components: { CustomTableRow },
  props: {
    tableConfig: Object, // Global settings (e.g., rowSelection, defaultSort)
    columnConfig: Array, // Column definitions (sortable, editable, etc.)
    data: Array, // Row data
  },
  setup() {
    const state = reactive({
      filterValues: {}, // Stores column-wise filter values
      filterInputValues: {}, // Stores input values for filters
      showFilterOptions: {}, // Stores visibility of filter options
      expandedRows: new Set(), // Stores IDs of expanded rows
      selectedRows: new Set(), // Stores IDs of selected rows
      globalSearch: "", // Stores global search value
      isEditing: false, // Editing state
      editedData: [], // Stores edited data
    });

    return {
      ...toRefs(state),
    };
  },
  computed: {
    // Filtered data based on filterValues and globalSearch
    filteredData() {
      return this.data.filter((row) => {
        const matchesGlobalSearch = Object.values(row).some((value) =>
          String(value).toLowerCase().includes(this.globalSearch.toLowerCase())
        );
        const matchesColumnFilters = Object.entries(this.filterValues).every(([key, value]) =>
          String(row[key]).toLowerCase().includes(value.toLowerCase())
        );
        return matchesGlobalSearch && matchesColumnFilters;
      });
    },
    // Sort and filter the data
    sortedFilteredData() {
      const { key, order } = this.tableConfig.defaultSort || {};
      if (!key) return this.filteredData;

      return [...this.filteredData].sort((a, b) => {
        const modifier = order === "asc" ? 1 : -1;
        return a[key] > b[key] ? modifier : -modifier;
      });
    },
  },
  methods: {
    handleSort(columnKey) {
      // eslint-disable-next-line vue/no-mutating-props
      if (this.tableConfig.defaultSort.key === columnKey) {
        // eslint-disable-next-line vue/no-mutating-props
        this.tableConfig.defaultSort.order =
          this.tableConfig.defaultSort.order === "asc" ? "desc" : "asc";
      } else {
        // eslint-disable-next-line vue/no-mutating-props
        this.tableConfig.defaultSort = { key: columnKey, order: "asc" };
      }
    },
    getSortIcon(columnKey) {
      if (this.tableConfig.defaultSort.key === columnKey) {
        return this.tableConfig.defaultSort.order === "asc" ? "↑" : "↓";
      }
      return "";
    },
    handleRowUpdate(updatedRow) {
      const findAndUpdateRow = (rows) => {
        for (let i = 0; i < rows.length; i++) {
          if (rows[i].id === updatedRow.id) {
            rows[i] = { ...rows[i], ...updatedRow };
            return true;
          } else if (rows[i].children) {
            if (findAndUpdateRow(rows[i].children)) return true;
          }
        }
        return false;
      };
      findAndUpdateRow(this.data);

      // Update the editedData array
      const editedRowIndex = this.editedData.findIndex(row => row.id === updatedRow.id);
      if (editedRowIndex !== -1) {
        this.editedData[editedRowIndex] = updatedRow;
      } else {
        this.editedData.push(updatedRow);
      }
    },
    handleSelectionChange(selectedRows) {
      this.$emit("selectionChanged", selectedRows);
    },
    toggleChildren(rowId) {
      if (this.expandedRows.has(rowId)) {
        this.expandedRows.delete(rowId);
      } else {
        this.expandedRows.add(rowId);
      }
    },
    handleSelectAll(event) {
      if (event.target.checked) {
        this.selectedRows = new Set(this.sortedFilteredData.map(row => row.id));
      } else {
        this.selectedRows.clear();
      }
      this.$emit("selectionChanged", Array.from(this.selectedRows));
    },
    toggleFilterOptions(columnKey) {
      this.showFilterOptions[columnKey] = !this.showFilterOptions[columnKey];
    },
    applyFilter(columnKey) {
      this.filterValues[columnKey] = this.filterInputValues[columnKey];
      this.showFilterOptions[columnKey] = false;
    },
    clearAllFilters() {
      this.filterValues = {};
      this.filterInputValues = {};
      this.globalSearch = "";
    },
    applyGlobalFilter() {
      // This method is triggered by the global search input
    },
    toggleEditOptions() {
      this.isEditing = !this.isEditing;
      if (this.isEditing) {
        this.$nextTick(() => {
          this.$refs.customTableRow.forEach(row => row.focusEditableFields());
        });
      } else {
        this.editedData = [];
      }
    },
    saveEdits() {
      // Create a local copy of the data prop
      const updatedData = this.data.map(row => {
        const editedRow = this.editedData.find(edited => edited.id === row.id);
        return editedRow ? editedRow : row;
      });

      // Emit the updated data to the parent component
      this.$emit("saveEdits", updatedData);

      // Clear the editedData array
      this.editedData = [];

      this.isEditing = false;
    },
    getRowClass(index) {
      let classes = [];
      if (this.tableConfig.stripedRows) {
        classes.push(index % 2 === 0 ? 'even-row' : 'odd-row');
      }
      if (this.selectedRows.has(index)) {
        classes.push('selected-row');
      }
      return classes;
    },
  },
};
</script>

<style>
.table {
  width: 100%;
  border-collapse: collapse;
}

.table-header {
  display: flex;
  background-color: #f4f4f4;
  font-weight: bold;
}

.header-cell {
  flex: 1;
  padding: 10px;
  cursor: pointer;
  position: relative;
}

.filter-options {
  position: absolute;
  top: 100%;
  left: 0;
  background: white;
  border: 1px solid #ccc;
  padding: 10px;
  z-index: 1;
}

.table-row-wrapper {
  border-bottom: 1px solid #ddd;
}

.super-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10px;
}

.global-search {
  flex: 1;
  margin-right: 10px;
}

.clear-filters {
  margin-right: 10px;
}

.edit-button {
  margin-right: 10px;
}

.even-row {
  background-color: #f9f9f9;
}

.odd-row {
  background-color: #e9e9e9;
}

.selected-row {
  background-color: #d3d3d3;
}
</style>
