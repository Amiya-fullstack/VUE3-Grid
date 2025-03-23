<template>
  <div id="app">
    <h1>Hierarchical Vue Table with Full Features</h1>
    <CustomTable
      :table-config="tableConfig"
      :column-config="columnConfig"
      :data="gridData"
      @update:tableConfig="updateTableConfig"
      @updateRow="handleRowUpdate"
      @selectionChanged="handleSelectionChange"
      @saveEdits="handleSaveEdits"
    />
  </div>
</template>

<script>
import CustomTable from "./components/CustomTable.vue";

export default {
  name: "App",
  components: {
    CustomTable,
  },
  data() {
    return {
      // **Table-level configuration**
      tableConfig: {
        rowSelection: true, // Enables row selection
        defaultSort: { key: "name", order: "asc" }, // Default sorting settings
        selectionAll: true, // Selects all rows
        tableFilter: true, // Enables table-level filter
        tableEdit: true, // Enables table-level editing
        stripedRows: true, // Applies striped row styling
        autoResizeColumns: true, // Enables auto-resizing of columns
      },

      // **Column-level configuration**
      columnConfig: [
        { key: "name", label: "Name", sortable: true, filterable: true, editable: true, type: "text" },
        { key: "age", label: "Age", sortable: true, filterable: false, editable: true, type: "number" },
        { key: "location", label: "Location", sortable: false, filterable: true, editable: false, type: "text" },
        { key: "status", label: "Status", sortable: false, filterable: true, editable: true, type: "select", options: ["Active", "Inactive"] },
        { key: "department", label: "Department", sortable: false, filterable: true, editable: true, type: "autocomplete", options: ["HR", "Engineering", "Marketing", "Sales"] },
      ],

      // **Hierarchical row data**
      gridData: [
        {
          id: 1,
          name: "Alice",
          age: 30,
          location: "New York",
          status: "Active",
          department: "HR",
          children: [
            {
              id: 11,
              name: "Alice Jr.",
              age: 10,
              location: "New York",
              status: "Active",
              department: "HR",
              children: [
                {
                  id: 111,
                  name: "Alice's Grandchild",
                  age: 2,
                  location: "New York",
                  status: "Active",
                  department: "HR",
                },
              ],
            },
          ],
        },
        {
          id: 2,
          name: "Bob",
          age: 25,
          location: "London",
          status: "Inactive",
          department: "Engineering",
          children: [
            {
              id: 21,
              name: "Bob Jr.",
              age: 5,
              location: "London",
              status: "Inactive",
              department: "Engineering",
            },
          ],
        },
        {
          id: 3,
          name: "Charlie",
          age: 35,
          location: "San Francisco",
          status: "Active",
          department: "Marketing",
        },
      ],
    };
  },
  methods: {
    updateTableConfig(newConfig) {
      this.tableConfig = newConfig;
    },
    handleRowUpdate(updatedRow) {
      // Update the gridData when a row is edited
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
      findAndUpdateRow(this.gridData);
    },
    handleSelectionChange(selectedRowIds) {
      console.log("Selected rows:", selectedRowIds);
      // Handle row selection logic (e.g., API call, UI updates, etc.)
    },
    handleSaveEdits(editedData) {
      editedData.forEach(updatedRow => {
        this.handleRowUpdate(updatedRow);
      });
      console.log("Saved edits:", editedData);
    },
  },
};
</script>

<style>
#app {
  font-family: Arial, sans-serif;
  text-align: center;
  margin: 20px;
}

h1 {
  margin-bottom: 20px;
}
</style>
