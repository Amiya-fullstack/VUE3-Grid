<template>
  <div>
    <div :class="['table-row', rowClass]">
      <div class="cell">
        <button @click="toggleChildren">{{ isExpanded ? '-' : '+' }}</button>
      </div>
      <div v-if="tableConfig.rowSelection" class="cell">
        <input type="checkbox" @change="handleSelectionChange" :checked="isSelected" />
      </div>
      <div
        class="cell"
        v-for="column in columnConfig"
        :key="column.key"
        :style="getColumnStyle(column.key)"
        :data-column-key="column.key"
      >
        <template v-if="isEditing && column.editable">
          <input v-if="column.type === 'text'" type="text" v-model="editValues[column.key]" ref="editableField" />
          <input v-if="column.type === 'number'" type="number" v-model="editValues[column.key]" ref="editableField" />
          <input v-if="column.type === 'date'" type="date" v-model="editValues[column.key]" ref="editableField" />
          <select v-if="column.type === 'select'" v-model="editValues[column.key]" ref="editableField">
            <option v-for="option in column.options" :key="option" :value="option">{{ option }}</option>
          </select>
          <v-autocomplete
            v-if="column.type === 'autocomplete'"
            v-model="editValues[column.key]"
            :items="column.options"
            :label="column.label"
            :clearable="false"
            ref="editableField"
          ></v-autocomplete>
        </template>
        <template v-else>
          {{ row[column.key] }}
        </template>
      </div>
    </div>
    <div v-if="isExpanded" class="child-rows">
      <CustomTableRow
        v-for="(child, childIndex) in row.children"
        :key="child.id"
        :row="child"
        :column-config="columnConfig"
        :table-config="tableConfig"
        :expanded-rows="expandedRows"
        :selected-rows="selectedRows"
        :is-editing="isEditing"
        :row-index="childIndex"
        :parent-index="rowIndex"
        @updateRow="$emit('updateRow', $event)"
        @selectionChanged="$emit('selectionChanged', $event)"
        @toggleChildren="$emit('toggleChildren', $event)"
      />
    </div>
  </div>
</template>

<script>
import { VAutocomplete } from 'vuetify/components'

export default {
  name: "CustomTableRow",
  components: {
    VAutocomplete,
  },
  props: {
    row: Object,
    columnConfig: Array,
    tableConfig: Object,
    expandedRows: Set,
    selectedRows: Set,
    isEditing: Boolean,
    rowIndex: Number,
    parentIndex: Number,
  },
  data() {
    return {
      isExpanded: false,
      editValues: { ...this.row },
      columnWidths: {},
    };
  },
  computed: {
    isSelected() {
      return this.selectedRows.has(this.row.id);
    },
    rowClass() {
      let classes = [];
      if (this.tableConfig.stripedRows) {
        const index = this.parentIndex !== undefined ? this.parentIndex + this.rowIndex : this.rowIndex;
        classes.push(index % 2 === 0 ? 'even-row' : 'odd-row');
      }
      if (this.isSelected) {
        classes.push('selected-row');
      }
      return classes;
    },
  },
  methods: {
    handleSelectionChange(event) {
      if (event.target.checked) {
        this.selectedRows.add(this.row.id);
      } else {
        this.selectedRows.delete(this.row.id);
      }
      this.$emit("selectionChanged", Array.from(this.selectedRows));
    },
    toggleChildren() {
      this.isExpanded = !this.isExpanded;
      this.$emit("toggleChildren", this.row.id);
    },
    saveEdit() {
      this.$emit("updateRow", { ...this.row, ...this.editValues });
      this.editValues = { ...this.row }; // Reset editValues after saving
    },
    getColumnStyle(columnKey) {
      if (this.tableConfig.autoResize && this.columnWidths[columnKey]) {
        return { width: this.columnWidths[columnKey] + 'px' };
      }
      return {};
    },
    autoResizeColumns() {
      if (!this.tableConfig.autoResize) return;

      this.$nextTick(() => {
        const columnWidths = {};
        this.$el.querySelectorAll('.cell').forEach(cell => {
          const columnKey = cell.getAttribute('data-column-key');
          if (columnKey) {
            columnWidths[columnKey] = Math.max(columnWidths[columnKey] || 0, cell.scrollWidth);
          }
        });
        this.columnWidths = columnWidths;
      });
    },
    focusEditableFields() {
      this.$nextTick(() => {
        this.$refs.editableField.forEach(field => field.focus());
      });
    },
  },
  mounted() {
    this.autoResizeColumns();
    window.addEventListener('resize', this.autoResizeColumns);
  },
  beforeUnmount() {
    window.removeEventListener('resize', this.autoResizeColumns);
  },
  watch: {
    isEditing(newVal) {
      if (newVal) {
        this.focusEditableFields();
      } else {
        this.editValues = { ...this.row };
      }
    },
    row: {
      handler() {
        this.autoResizeColumns();
      },
      deep: true,
    },
  },
};
</script>

<style>
.table-row {
  display: flex;
  align-items: center;
}

.cell {
  flex: 1;
  padding: 10px;
}

.child-rows {
  padding-left: 20px;
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
