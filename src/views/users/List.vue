<template>
  <v-card elevation="24" :disabled="isLoading">
    <v-card-title>
      <v-row dense>
        <v-col cols="10">
          <CardTitle :text="route.meta.title" :icon="route.meta.icon" />
        </v-col>
        <v-col cols="2" class="text-right">
          <div class="d-flex justify-end" style="gap: 10px">
            <v-btn icon variant="flat" size="x-small" color="warning" @click="downloadUsers">
              <v-icon>mdi-list-box</v-icon>
              <v-tooltip activator="parent" location="bottom">Listado de empleados</v-tooltip>
            </v-btn>
            <v-btn
              icon
              variant="flat"
              size="x-small"
              color="success"
              :to="{ name: `${routeName}/store` }"
            >
              <v-icon>mdi-plus</v-icon>
              <v-tooltip activator="parent" location="bottom">Agregar</v-tooltip>
            </v-btn>
          </div>
        </v-col>
      </v-row>
    </v-card-title>

    <v-card-text>
      <v-row dense>
        <v-col cols="12" md="9" class="pb-0">
          <v-row dense>
            <v-col v-if="store.getAuth?.user?.role_id === 1" cols="12" md="3" class="pb-0">
              <v-select
                label="Mostrar"
                v-model="active"
                variant="outlined"
                density="compact"
                :items="activeOptions"
                item-title="name"
                item-value="id"
                :disabled="!isItemsEmpty"
              />
            </v-col>
            <v-col cols="12" md="3" class="pb-0">
              <v-select
                label="Filtro"
                v-model="filter"
                variant="outlined"
                density="compact"
                :items="filterOptions"
                item-title="name"
                item-value="id"
                :disabled="!isItemsEmpty"
              />
            </v-col>
          </v-row>
        </v-col>

        <v-col cols="12" md="3" class="pb-0">
          <v-text-field
            label="Buscar"
            v-model="search"
            type="text"
            variant="outlined"
            density="compact"
            append-inner-icon="mdi-magnify"
            :disabled="isItemsEmpty"
          />
        </v-col>

        <v-col cols="12">
          <v-btn
            block
            size="small"
            :color="isItemsEmpty ? 'info' : 'grey-darken-1'"
            :loading="isItemsEmpty && isLoading"
            @click.prevent="isItemsEmpty ? getItems() : (items = [])"
          >
            {{ isItemsEmpty ? 'Aplicar' : 'Cambiar' }} filtros
            <v-icon right>mdi-filter</v-icon>
          </v-btn>
        </v-col>

        <v-col cols="12">
          <v-data-table
            density="compact"
            :items="items"
            :headers="headers"
            :search="search"
            :items-per-page="15"
            :loading="isLoading"
          >
            <template #item.key="{ item }">
              <b>{{ item.key + 1 }}</b>
            </template>

            <template #item.email_verified_at="{ item }">
              <v-icon size="x-small" :color="item.email_verified_at ? 'info' : ''">
                mdi-checkbox-blank-circle{{ item.email_verified_at ? '' : '-outline' }}
              </v-icon>
            </template>

            <template #item.action="{ item }">
              <div class="text-right">
                <v-btn
                  icon
                  variant="text"
                  size="x-small"
                  :color="item.active ? '' : 'error'"
                  :to="{ name: `${routeName}/show`, params: { id: getEncodeId(item.id) } }"
                >
                  <v-icon>mdi-eye</v-icon>
                  <v-tooltip activator="parent" location="left">Detalle</v-tooltip>
                </v-btn>
              </div>
            </template>
          </v-data-table>
        </v-col>
      </v-row>
    </v-card-text>
  </v-card>
</template>

<script setup>
// Importaciones de librerías externas
import { ref, inject, computed, onMounted } from 'vue'
import { useRouter, useRoute } from 'vue-router'
import axios from 'axios'

// Importaciones internas del proyecto
import { useStore } from '@/store'
import { URL_API } from '@/utils/config'
import { getHdrs, getErr, getRsp } from '@/utils/http'
import { getEncodeId } from '@/utils/coders'

// Componentes
import CardTitle from '@/components/CardTitle.vue'

// Constantes fijas
const routeName = 'users'

// Estado y referencias
const alert = inject('alert')
const store = useStore()
const router = useRouter()
const route = useRoute()

// Estado reactivo
const isLoading = ref(false)
const items = ref([])
const isItemsEmpty = computed(() => items.value.length === 0)
const headers = ref([])
const search = ref('')
const active = ref(1)
const activeOptions = ref([])
const filter = ref(0)
const filterOptions = ref([])

// Cargar registros
const getItems = async () => {
  isLoading.value = true
  items.value = []

  try {
    const endpoint = `${URL_API}/${routeName}?active=${active.value}&filter=${filter.value}`
    const response = await axios.get(endpoint, getHdrs(store.getAuth?.token))
    items.value = getRsp(response).data.items
  } catch (err) {
    alert?.show('red-darken-1', getErr(err))
  } finally {
    isLoading.value = false
  }
}

const downloadUsers = async () => {
  isLoading.value = true
  try {
    const endpoint = `${URL_API}/${routeName}/file/json`
    const response = await axios.get(endpoint, getHdrs(store.getAuth?.token))
    const data = getRsp(response).data

    if (typeof data === 'string') {
      const byteCharacters = atob(data)
      const byteNumbers = new Array(byteCharacters.length)

      for (let i = 0; i < byteCharacters.length; i++) {
        byteNumbers[i] = byteCharacters.charCodeAt(i)
      }

      const byteArray = new Uint8Array(byteNumbers)
      const blob = new Blob([byteArray], { type: 'application/octet-stream' })

      const link = document.createElement('a')
      link.href = URL.createObjectURL(blob)
      link.download = `usuarios_${new Date().toISOString().slice(0, 10)}.svr`
      document.body.appendChild(link)
      link.click()
      document.body.removeChild(link)

      alert?.show('success', 'Archivo descargado correctamente')
    } else {
      throw new Error('Formato de datos no válido')
    }
  } catch (err) {
    alert?.show('red-darken-1', getErr(err) || 'Error al descargar el archivo')
  } finally {
    isLoading.value = false
  }
}

// Inicializar
onMounted(() => {
  headers.value = [
    { title: '#', key: 'key', filterable: false, sortable: false, width: 60 },
    { title: 'Nombre', key: 'name' },
    { title: 'E-mail', key: 'email' },
    { title: 'Rol', key: 'role.name' },
    { title: 'ID', key: 'uiid' },
    { title: 'Verif.', key: 'email_verified_at' },
    { title: '', key: 'action', filterable: false, sortable: false, width: 60 },
  ]

  activeOptions.value = [
    { id: 1, name: 'ACTIVOS' },
    { id: 0, name: 'INACTIVOS' },
  ]

  filterOptions.value = [{ id: 0, name: 'TODOS' }]

  getItems()
})
</script>
