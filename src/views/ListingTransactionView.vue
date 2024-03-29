<script setup>
import { computed, onMounted, ref } from 'vue';
import { useRouter } from 'vue-router';
import { useStore } from 'vuex';
import MerchantWaiting from '../components/MerchantWaiting.vue';
import TransactionStatusChip from '../components/TransactionStatusChip.vue';
import formatDate from '../utils/formatDate';

const transactions = ref([]);
const store = useStore();
const router = useRouter();

const userId = computed(() => store.state.id);
const isAdmin = computed(() => store.state.isAdmin);
const isApproved = computed(() => store.state.isApproved);
const token = computed(() => store.state.token);
const merchantId = computed(() => {
  if (store.getters.getSelectedMerchant) {
    return store.getters.getSelectedMerchant;
  }
  return store.state.merchantId;
});

const searchInput = ref('');
const searchByStatus = ref(null);

const selectItems = [
  { text: 'Tous', value: null },
  { text: 'En attente de paiement', value: 'created' },
  { text: 'Autorisée', value: 'authorized' },
  { text: 'Partiellement payée', value: 'partial_captured' },
  { text: 'Payé', value: 'captured' },
  { text: 'En attente de remboursement', value: 'waiting_refund' },
  { text: 'Partiellement remboursée', value: 'partial_refunded' },
  { text: 'Remboursée', value: 'refunded' },
];

// Function to fetch merchants
const getAllTransactions = async () => {
  const response = await fetch(`${import.meta.env.VITE_SERVER_URL}/api/transactions`, {
    method: 'GET',
    headers: {
      'Content-Type': 'application/json',
      token: token.value,
    },
    credentials: 'include',
  });

  if (response.ok) {
    const data = await response.json();

    transactions.value = data;
  } else if (response.status === 401) {
    console.error('You are not authorized.');
    router.push('/login');
  } else {
    console.error('An error occurred.');
  }
};

const getTransactions = async (id) => {
  const response = await fetch(`${import.meta.env.VITE_SERVER_URL}/api/merchants/${id}/transactions`, {
    method: 'GET',
    headers: {
      'Content-Type': 'application/json',
      token: token.value,
    },
    credentials: 'include',
  });
  if (response.ok) {
    const data = await response.json();
    transactions.value = data;
  } else {
    console.error('An error occurred.');
  }
};

const goToTransactionDetails = async (id) => {
  router.push(`/listing-transaction/${id}`);
};

const searchFilter = () => {
  const searchInputValue = searchInput.value.toLowerCase().trim();

  const byAmountAndCurrency = transactions.value.filter((transaction) => {
    const findedAmount = transaction.amount.toString().includes(searchInputValue);
    // const findedStatus = transaction.status.toLowerCase().includes(searchByStatus.value);
    const findedCurrency = transaction.currency.toLowerCase().includes(searchInputValue);

    return findedAmount || findedCurrency;
  });

  const toto = byAmountAndCurrency.filter((transaction) => searchByStatus.value !== null ? transaction.status === searchByStatus.value : true);

  return toto;
};

onMounted(async () => {
  if (isAdmin.value && store.getters.getSelectedMerchant === null) {
    await getAllTransactions();
  } else if (isApproved.value || store.getters.getSelectedMerchant) {
    await getTransactions(merchantId.value);
  } else if (userId.value === null) {
    router.push('/login');
  }
});
</script>

<template>
  <div class=" m-6 relative overflow-x-auto shadow-md sm:rounded-lg" v-if="isAdmin || isApproved">
    <h2 class="my-2 text-2xl">Liste des transactions</h2>
    <div class="my-2">
      <label class="block text-sm font-medium text-gray-700 mb-3">Rechercher par amount - currency</label>
      <div class="flex space-x-2">
        <input v-model="searchInput" type="text" placeholder="Rechercher" class="px-2 py-1 border rounded input-search">
        <v-select
            label="Statut"
            :items="selectItems"
            :item-title="item => item.text"
            :item-value="item => item.value"
            chips="true"
            v-model="searchByStatus"
            variant="outlined"
            density="compact"
        />
      </div>
    </div>
    <div v-if="isAdmin && !store.getters.getSelectedMerchant">
      <table class="w-full text-sm text-left text-gray-500 dark:text-gray-400" v-if="searchFilter().length > 0">
        <thead class="text-xs text-gray-700 uppercase bg-gray-50 dark:bg-gray-700 dark:text-gray-400">
        <tr>
          <th scope="col" class="px-6 py-3">Amount</th>
          <th scope="col" class="px-6 py-3">Status</th>
          <th scope="col" class="px-6 py-3">Devise</th>
          <th scope="col" class="px-6 py-3">Marchand</th>
          <th scope="col" class="px-6 py-3">Utilisateur</th>
          <th scope="col" class="px-6 py-3">Date</th>
          <th scope="col" class="px-6 py-3">Action</th>
        </tr>
        </thead>
        <tbody>
        <tr v-for="transaction in searchFilter()" v-bind:key="transaction" class="bg-white border-b dark:bg-gray-800
            dark:border-gray-700 hover:bg-gray-50 dark:hover:bg-gray-600">
          <td class="px-6 py-4">{{ transaction.amount }}</td>
          <td class="px-6 py-4"><TransactionStatusChip :transaction-status="transaction.status" /></td>
          <td class="px-6 py-4">{{ transaction.currency }}</td>
          <td class="px-6 py-4">{{ transaction.merchantId }}</td>
          <td class="px-6 py-4">{{ transaction.userId }}</td>
          <td class="px-6 py-4">{{ formatDate(transaction.createdDate) }}</td>
          <td class="px-6 py-4">
            <button class="px-4 py-2 font-semibold text-white bg-green-500 rounded hover:bg-green-700"
                    @click="goToTransactionDetails(transaction.transactionId)">
              Voir détails
            </button>
          </td>
        </tr>
        </tbody>
      </table>
      <div v-else>
        <p>Vous n'avez aucune transaction / Essayez un autre filtre </p>
      </div>
    </div>
    <div v-else-if="isApproved || merchantId">
      <table class="w-full text-sm text-left text-gray-500 dark:text-gray-400" v-if="searchFilter().length > 0">
        <thead class="text-xs text-gray-700 uppercase bg-gray-50 dark:bg-gray-700 dark:text-gray-400">
        <tr>
          <th scope="col" class="px-6 py-3">Amount</th>
          <th scope="col" class="px-6 py-3">Status</th>
          <th scope="col" class="px-6 py-3">Devise</th>
          <th scope="col" class="px-6 py-3">Utilisateur</th>
          <th scope="col" class="px-6 py-3">Date</th>
          <th scope="col" class="px-6 py-3">Action</th>

        </tr>
        </thead>
        <tbody>
        <tr v-for="transaction in searchFilter()" v-bind:key="transaction" class="bg-white border-b dark:bg-gray-800
            dark:border-gray-700 hover:bg-gray-50 dark:hover:bg-gray-600">
          <td class="px-6 py-4">{{ transaction.amount }}</td>
          <td class="px-6 py-4"><TransactionStatusChip :transaction-status="transaction.status" /></td>
          <td class="px-6 py-4">{{ transaction.currency }}</td>
          <td class="px-6 py-4">{{ transaction.userId }}</td>
          <td class="px-6 py-4">{{ formatDate(transaction.createdAt) }}</td>
          <td class="px-6 py-4">
            <button class="px-4 py-2 font-semibold text-white bg-green-500 rounded hover:bg-green-700"
                    @click="goToTransactionDetails(transaction.id)">
              Voir détails
            </button>
          </td>
        </tr>
        </tbody>
      </table>
      <div v-else>
        <p>Vous n'avez aucune transaction / Changez votre filtre</p>
      </div>
    </div>
  </div>
  <div class="flex flex-col items-center mb-4" v-else>
    <MerchantWaiting />
  </div>
</template>

<style scoped>
.input-search{
  margin-bottom: 22px !important;
}
</style>
