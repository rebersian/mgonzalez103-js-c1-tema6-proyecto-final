<template>
  <div class="container py-5">
    <section class="col-md-6 col-lg-5 mx-auto border rounded-3 overflow-hidden">
      <!-- Cabecera gris -->
      <div class="bg-body-tertiary py-4 px-4 text-center border-bottom">
        <h1 class="display-6 fw-bold mb-0">Perfil de usuario</h1>
      </div>

      <!-- Contenido blanco -->
      <div v-if="user" class="bg-white p-4">
        <div class="card-body py-2">
          <p><strong>Email: </strong>{{ user.email }}</p>
          <p><strong>Nombre completo: </strong> {{ user.fullName }}</p>
          <p><strong>ID: </strong>{{ user.id }}</p>
          <p><strong>Activo: </strong> {{ user.isActive ? 'Si' : 'No' }}</p>
        </div>

        <div class="py-2">
          <button class="btn btn-outline-danger btn-sm w-100" type="button" @click="logout">
            Cerrar sesión
          </button>
        </div>
      </div>
      <div v-else class="bg-white p-4">
        <p><strong>No se han encontrado datos del usuario </strong></p>
      </div>
    </section>
  </div>
</template>

<script setup lang="ts">
import { useToast } from '@/common/composables/useToast';
import { clearAuth } from '@/modules/auth/utils/token';
import { useRouter } from 'vue-router';
import type { User } from '@/modules/auth/models/user.model';
import { onMounted, ref } from 'vue';
import { getInfoUser } from '@/modules/auth/services/auth.service';

const router = useRouter();
const toast = useToast();
const user = ref<User>();

function logout() {
  clearAuth();
  toast.info('Sesión cerrada');
  router.push('/login');
}
const getUsuario = async () => {
  try {
    user.value = (await getInfoUser()).user;
  } catch (error) {
    console.error('Error obteniendo los datos del usuario', error);
  }
};
onMounted(getUsuario);
</script>
