<template lang="pug">
q-page.flex
  q-form.q-ml-lg.q-mt-lg(@submit="createSession")
    .text-h3 Создание новой сессии
    q-input(v-model='clientName', label='Ваш ник', stack-label='', :dense='dense')
    q-input(v-model='floors', label='Кол-во этажей', stack-label='', :dense='dense')
    q-input(v-model='timeFactor', label='Коэффициент замедления (по умолчанию - 1.0)', stack-label='', :dense='dense')
    q-btn.q-mt-sm(type="submit" color="primary") Создать
</template>

<script setup>
import { ref, watch } from "vue";
import { api } from "boot/axios";

const clientName = ref("");
const floors = ref("");
const timeFactor = ref("");
const dense = ref(false);

const createSession = async () => {
  const res = await api.post("/newSession", null, {
    params: {
      clientName: clientName.value,
      floors: floors.value,
      timeFactor: timeFactor.value || "1.0"
    }
  });
  console.log(res);
};

// watch(() => [clientName.value, floors.value, cef.value]);
</script>
