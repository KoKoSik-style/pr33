<script setup>
import { ref, reactive } from 'vue';
import axios from 'axios';

const isLoading = ref(false);
const successMessage = ref('');
const generalError = ref(null);
const isSubmitted = ref(false); // Флаг: была ли попытка отправки (для живой валидации)

// Данные формы
const form = reactive({
  name: '',
  email: '',
  password: '',
  confirmPassword: ''
});

// Ошибки валидации
const errors = reactive({
  name: null,
  email: null,
  password: null,
  confirmPassword: null
});

// Функция валидации конкретного поля (для @blur и @input)
const validateField = (field) => {
  errors[field] = null;

  if (field === 'name') {
    if (!form.name || form.name.trim().length < 2) {
      errors.name = 'Имя должно быть не короче 2 символов.';
    }
  }

  if (field === 'email') {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    if (!form.email || !emailRegex.test(form.email)) {
      errors.email = 'Введите корректный email.';
    }
  }

  if (field === 'password') {
    // Проверка: 8+ символов, строчные, прописные, цифры, спецсимволы
    const hasLength = form.password.length >= 8;
    const hasLower = /[a-z]/.test(form.password);
    const hasUpper = /[A-Z]/.test(form.password);
    const hasNumber = /[0-9]/.test(form.password);
    const hasSpecial = /[^A-Za-z0-9]/.test(form.password);

    if (!form.password) {
      errors.password = 'Пароль обязателен.';
    } else if (!hasLength || !hasLower || !hasUpper || !hasNumber || !hasSpecial) {
      errors.password = 'Пароль должен содержать минимум 8 символов, включая строчные и прописные буквы, цифры и спецсимволы.';
    }
  }

  if (field === 'confirmPassword') {
    if (form.password !== form.confirmPassword) {
      errors.confirmPassword = 'Пароли не совпадают.';
    } else if (!form.confirmPassword && form.password) {
      errors.confirmPassword = 'Подтвердите пароль.';
    }
  }
};

// Валидация всей формы перед отправкой
const validateAll = () => {
  validateField('name');
  validateField('email');
  validateField('password');
  validateField('confirmPassword');
  return !Object.values(errors).some(error => error !== null);
};

// Обработчик ввода (живая валидация после первой отправки)
const onInput = (field) => {
  if (isSubmitted.value) {
    validateField(field);
  }
};

const handleSubmit = async () => {
  isSubmitted.value = true; // Включаем живую валидацию
  successMessage.value = '';
  generalError.value = null;

  if (!validateAll()) {
    return;
  }

  isLoading.value = true;

  try {
    const payload = {
      name: form.name.trim(),
      email: form.email.trim(),
      password: form.password
      // confirmPassword обычно на бэкенд не отправляют
    };

    // Отправка через Axios
    const response = await axios.post('https://jsonplaceholder.typicode.com/users', payload);
    
    // Очистка формы после успеха
    form.name = '';
    form.email = '';
    form.password = '';
    form.confirmPassword = '';
    isSubmitted.value = false;
    
    successMessage.value = `Регистрация успешна! ID пользователя: ${response.data.id || 'демо'}`;
  } catch (err) {
    // Обработка ошибок Axios
    if (err.code === 'ERR_NETWORK') {
      generalError.value = 'Ошибка сети. Проверьте интернет-соединение.';
    } else if (err.response) {
      // Сервер ответил с кодом ошибки (4xx, 5xx)
      generalError.value = `Ошибка сервера: ${err.response.status} ${err.response.statusText || ''}`;
    } else {
      generalError.value = 'Произошла неизвестная ошибка при отправке.';
    }
  } finally {
    isLoading.value = false;
  }
};
</script>

<template>
  <div class="form-page">
    <h1>Регистрация</h1>

    <form @submit.prevent="handleSubmit" class="registration-form">
      <!-- Имя -->
      <div class="form-group">
        <label for="name">Имя</label>
        <input
          id="name"
          v-model="form.name"
          type="text"
          class="form-input"
          :class="{ 'input-error': errors.name }"
          @blur="validateField('name')"
          @input="onInput('name')"
        />
        <span v-if="errors.name" class="error-message">{{ errors.name }}</span>
      </div>

      <!-- Email -->
      <div class="form-group">
        <label for="email">Email</label>
        <input
          id="email"
          v-model="form.email"
          type="email"
          class="form-input"
          :class="{ 'input-error': errors.email }"
          @blur="validateField('email')"
          @input="onInput('email')"
        />
        <span v-if="errors.email" class="error-message">{{ errors.email }}</span>
      </div>

      <!-- Пароль -->
      <div class="form-group">
        <label for="password">Пароль</label>
        <input
          id="password"
          v-model="form.password"
          type="password"
          class="form-input"
          :class="{ 'input-error': errors.password }"
          @blur="validateField('password')"
          @input="onInput('password')"
        />
        <span v-if="errors.password" class="error-message">{{ errors.password }}</span>
      </div>

      <!-- Подтверждение пароля -->
      <div class="form-group">
        <label for="confirmPassword">Подтверждение пароля</label>
        <input
          id="confirmPassword"
          v-model="form.confirmPassword"
          type="password"
          class="form-input"
          :class="{ 'input-error': errors.confirmPassword }"
          @blur="validateField('confirmPassword')"
          @input="onInput('confirmPassword')"
        />
        <span v-if="errors.confirmPassword" class="error-message">{{ errors.confirmPassword }}</span>
      </div>

      <!-- Общее сообщение об ошибке -->
      <div v-if="generalError" class="general-error">{{ generalError }}</div>

      <!-- Кнопка отправки -->
      <button type="submit" class="btn-submit" :disabled="isLoading">
        {{ isLoading ? 'Отправка...' : 'Зарегистрироваться' }}
      </button>
    </form>

    <!-- Сообщение об успехе -->
    <div v-if="successMessage" class="success-message">
      {{ successMessage }}
    </div>
  </div>
</template>

<style scoped>
.form-page {
  max-width: 500px;
  margin: 40px auto;
  padding: 20px;
  font-family: sans-serif;
}
.registration-form {
  display: flex;
  flex-direction: column;
  gap: 16px;
}
.form-group {
  display: flex;
  flex-direction: column;
  gap: 6px;
}
label {
  font-weight: bold;
  font-size: 14px;
}
.form-input {
  padding: 10px;
  border: 1px solid #ccc;
  border-radius: 6px;
  font-size: 16px;
  transition: border-color 0.2s;
}
.form-input:focus {
  outline: none;
  border-color: #42b883;
}
.input-error {
  border-color: #e74c3c;
  background-color: #fdf2f2;
}
.error-message {
  color: #e74c3c;
  font-size: 13px;
}
.general-error {
  background-color: #fce4e4;
  color: #c0392b;
  padding: 12px;
  border-radius: 6px;
  border: 1px solid #f5c6cb;
  font-size: 14px;
}
.btn-submit {
  padding: 12px;
  background-color: #42b883;
  color: white;
  border: none;
  border-radius: 6px;
  font-size: 16px;
  cursor: pointer;
  transition: background-color 0.2s;
}
.btn-submit:hover:not(:disabled) {
  background-color: #369e6e;
}
.btn-submit:disabled {
  background-color: #a0a0a0;
  cursor: not-allowed;
}
.success-message {
  margin-top: 20px;
  padding: 12px;
  background-color: #d4edda;
  color: #155724;
  border: 1px solid #c3e6cb;
  border-radius: 6px;
  text-align: center;
}
</style>