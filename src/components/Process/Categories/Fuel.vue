<template>
<div>
  <el-dialog title="Crear combustible"
             :visible.sync="dialogCreateVisible"
             width="30%">
    <div class="row">
      <div class="col-md-12">
        <label class="control-label">Nombre del combustible</label>
        <div class="form-group required">
          <input v-model="nameFuel"
                 type="text"
                 name="name"
                 class="form-control"
                 placeholder="Ingrese el nombre del combustible">
        </div>
      </div>
    </div>
    <span slot="footer"
          class="dialog-footer">
    <el-button @click="dialogCreateVisible = false">Cancelar</el-button>
    <el-button  @click="addRow()" :disabled="!nameFuel" type="primary">Continuar</el-button>
  </span>
  </el-dialog>
  <!-- MENU DE COMPONENTE -->
  <div class="row">
    <div class="col-md-12"
         style="margin-bottom:10px;">
      <button v-if="!showUpload"
              @click="openModal"
              style="margin-left:10px;"
              class="btn btn-sm btn-warning float-right"><i class="icon plus"></i>Agregar</button>
      <button @click="deleteRow(selectedRow)"
              v-if="selectedRow && !showUpload"
              style="margin-left:10px;"
              class="btn btn-sm btn-danger float-right"><i class="icon trash"></i>Eliminar seleccionado (id:{{selectedRow.id}})</button>
      <!--

      <button @click="download()"
              style="margin-left:10px;"
              class="btn btn-sm btn-dark float-right"><i class="icon download"></i>Descargar plantilla</button>
      <button @click="showUpload = true"
              class="btn btn-sm btn-primary float-right"><i class="icon upload"></i>Cargar plantilla</button>

      -->
    </div>
  </div>
  <!-- DATA GRID -->
  <div v-if="!showUpload && this.rows.length > 0">
    <ReactDataGrid :enableCellSelect="true"
                   :enableDragAndDrop="true"
                   key="Areas"
                   :columns="columns"
                   @rowGetter="rowGetter"
                   :rowsCount="countRows()"
                   @onRowClick="clickedRow"
                   :minHeight="500"
                   @onGridRowsUpdated="handleGridRowsUpdated"
                   ;></ReactDataGrid>
  </div>
  <div v-if="this.rows.length < 1">
    <center>
      <h2>No existen combustibles</h2>
      <img style="margin-bottom:10px;"
           width="100"
           src="/static/truck.svg">
      <hr>
      <button @click="openModal()"
              class="btn btn-lg btn-primary"><i class="icon add"></i>Agregar nuevo</button>
    </center>
  </div>
  <!-- UPLOAD -->
  <div v-if="showUpload"
       style="margin-top:100px;">
    <center>
      <el-upload class="upload-demo"
                 drag
                 :on-success="successUpload"
                 :action="uploadPath"
                 multiple>
        <i class="el-icon-upload"></i>
        <div class="el-upload__text">Suelta tu archivo aquí o <em>haz clic para cargar</em></div>
        <div slot="tip"
             class="el-upload__tip">Solo archivos excel</div>
      </el-upload>
      <hr>
      <button @click="showUpload = false"
              class="btn btn-outline-danger btn-lg">
        Cancelar subida</button>
    </center>
  </div>
</div>

</template>

<script>
import ReactDataGrid from 'react-data-grid';
import api from '../../../api/process.js';
import _ from 'lodash';
export default {
  /* Nombre del componente */
  name: 'Fuel',
  mounted() {
    if (!this.$auth.check([
        'fuels'
      ])) {
      this.$message({
        message: 'No tienes permisos',
        type: 'error'
      })
      this.$parent.goToCategory(1)
      return
    }
  },
  /* Propiedades */
  props: [
    'data'
  ],
  computed: {
    rows: function(e) {
      return this.data.fuels
    }
  },
  /* Componentes usuados */
  components: {
    ReactDataGrid
  },
  /* Metodos publicos */
  methods: {
    openModal() {
      this.dialogCreateVisible = true
      this.nameFuel = 'Combustible ' + parseInt(this.rows.length + 1, 10)
    },
    /* Mensaje en caso de que el archivo suba correctamente */
    successUpload(response) {
      this.rows = response
      this.rows.splice(0, 0)
      this.showUpload = false
      this.$message({
        message: 'Importación realizada correctamente',
        type: 'success'
      })
      this.$bus.emit('getModulesData', [
        {
          reference: 'fuels',
          url: 'getCombustiblesByProcessId'
        }
      ])
    },
    /* 
     * Permite agregar un registro
    */
    addRow() {
      this.dialogCreateVisible = false
      this.selectedRow = null
      api.createFuel({
        name: this.nameFuel,
        process_id: this.$route.params.id
      }).then((r) => {
        if (r.status === 200) {
          this.$bus.emit('getModulesData', [
            {
              reference: 'fuels',
              url: 'getCombustiblesByProcessId'
            }
          ])
        } else {
          this.$message({
            message: 'Error al agregar fila.',
            type: 'error'
          })
        }
      }).catch(() => {
        console.log('error')
      })
    },
    /* 
     * Permite eliminar un registro
    */
    deleteRow(row) {
      api.deleteFuel(row.id).then((r) => {

        if (_.isNil(r.response)) {
          this.rows.splice(this.rows.indexOf(row), 1)
          this.selectedRow = null
          this.$bus.emit('getModulesData', [
            {
              reference: 'fuels',
              url: 'getCombustiblesByProcessId'
            }
          ])

          this.$message({
            message: 'Combustible eliminado correctamente',
            type: 'success'
          })

        } else {
          this.$message({
            message: 'No se ha podido eliminar el registro, esta siendo usado en otra dependencia.',
            type: 'error'
          })
        }
      }).catch((err) => {
        console.log(err)
      })
    },
    /* 
     * Permite seleccionar un registro
    */
    clickedRow(row) {
      this.selectedRow = this.rows[row]
    },
    /* 
     * Permite descargar una matriz
    */
    download() {
      var ids = []
      ids.unshift([
        'ID',
        'name'
      ])

      api.donwloadExcelAreas(this.$route.params.id).then((response) => {

        var a = document.createElement('a')
        a.href = response.data.file
        a.download = response.data.name
        document.body.appendChild(a)
        a.click()
        a.remove()

      })

    },
    /* 
     * Permite obtener un row especifico
    */
    rowGetter(i) {
      return this.rows[i]
    },
    /* 
     * Permite actualizar registros
    */
    handleGridRowsUpdated(data) {

      for (let i = data.fromRow; i <= data.toRow; i++) {
        let rowToUpdate = this.rows[i]
        let updatedRow = _.extend(rowToUpdate, data.updated)
        api.updateFuel(updatedRow.id, updatedRow).then((r) => {
          if (r.status === 200) {
            this.rows[i] = updatedRow
            this.$bus.emit('getModulesData', [
              {
                reference: 'fuels',
                url: 'getCombustiblesByProcessId'
              }
            ])
          } else {

            this.rows[i].name = 'Inserte nuevo valor'
            this.$forceUpdate()
            this.$message({
              message: 'Error al modificar, el combustible ya existe.',
              type: 'error'
            })

          }
        }).catch((e) => {
          console.log(e)
        })
      }
    },
    countRows() {
      return this.rows.length
    }
  },
  data() {
    return {
      nameFuel: '',
      dialogCreateVisible: false,
      uploadPath: window.pathBase + ':' + window.portBaseLaravel + '/api/v1/fuel/importExcel/' + this.$route.params.id,
      showUpload: false, // determina si muestra el upload
      selectedRow: null, // determina si la fila esta seleccionada 
      columns: [ // nombre de las columnas 
        {
          key: 'name',
          name: 'Nombre',
          editable: true
        }
      ]
    }
  }
}

</script>
