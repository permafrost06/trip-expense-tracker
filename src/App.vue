<script setup lang="ts">
import { ref, computed, onMounted, watch } from 'vue';

interface ExpenseItem {
    description: string;
    amount: number;
    paidFor: string[];
}

interface Expense {
  description: string;
  paidBy: string;
  amount: number;
    paidFor: string[];
    items?: ExpenseItem[];  // Optional itemized breakdown
}

interface Person {
    name: string;
    count: number;  // Number of people this entry represents
}

interface TripData {
    id: string;
    name: string;
    people: Person[];
    expenses: Expense[];
}

const defaultPeople: Person[] = [
    { name: 'Zahin+Samiha', count: 2 },
    { name: 'Zerin', count: 1 },
    { name: 'Dipra', count: 1 },
];

const defaultExpenses: Expense[] = [
    { description: 'Bargee Lake Valley Resort Booking Advance', paidBy: 'Zahin+Samiha', amount: 1000, paidFor: ['Zahin+Samiha', 'Zerin', 'Dipra'] },
    { description: 'Bargee Lake Valley Resort Booking Advance', paidBy: 'Zerin', amount: 1500, paidFor: ['Zahin+Samiha', 'Zerin', 'Dipra'] },
    { description: 'Dhaka to Rangamati Bus', paidBy: 'Zahin+Samiha', amount: 1300, paidFor: ['Zahin+Samiha', 'Zerin', 'Dipra'] },
    { description: 'Rangamati to Dhaka Bus', paidBy: 'Zerin', amount: 2060, paidFor: ['Zahin+Samiha', 'Zerin', 'Dipra'] },
    { description: 'Breakfast at Rangamati', paidBy: 'Zerin', amount: 220, paidFor: ['Zahin+Samiha', 'Zerin', 'Dipra'] },
    { description: 'Water', paidBy: 'Zahin+Samiha', amount: 125, paidFor: ['Zahin+Samiha', 'Zerin', 'Dipra'] },
    { description: 'Shuvolong waterfall entry ticket', paidBy: 'Zahin+Samiha', amount: 50, paidFor: ['Zahin+Samiha', 'Zerin', 'Dipra'] },
    { description: 'Polwel Park entry ticket', paidBy: 'Dipra', amount: 160, paidFor: ['Zahin+Samiha', 'Zerin', 'Dipra'] },
    { description: 'Boat fare', paidBy: 'Zahin+Samiha', amount: 2000, paidFor: ['Zahin+Samiha', 'Zerin', 'Dipra'] },
    { description: 'Boat fare', paidBy: 'Zerin', amount: 500, paidFor: ['Zahin+Samiha', 'Zerin', 'Dipra'] },
    {
        description: 'Lunch at Bargee',
        paidBy: 'Dipra',
        amount: 1410,
        paidFor: ['Zahin+Samiha', 'Zerin', 'Dipra'],
        items: [
            { description: 'Rice', amount: 280, paidFor: ['Zahin+Samiha', 'Zerin', 'Dipra'] },
            { description: 'Sonali Chicken', amount: 250, paidFor: ['Zahin+Samiha'] },
            { description: 'Deshi Chicken', amount: 250, paidFor: ['Dipra'] },
            { description: 'Dal', amount: 210, paidFor: ['Zahin+Samiha', 'Zerin', 'Dipra'] },
            { description: 'Vegetable', amount: 80, paidFor: ['Zahin+Samiha'] },
            { description: 'Vegetable', amount: 80, paidFor: ['Dipra'] },
            { description: 'Chapila fry', amount: 120, paidFor: ['Zahin+Samiha'] },
            { description: 'Vorta', amount: 100, paidFor: ['Zerin'] },
            { description: 'Water', amount: 40, paidFor: ['Zahin+Samiha', 'Zerin', 'Dipra'] },
        ]
    },
    { description: 'Snacks for night', paidBy: 'Zahin+Samiha', amount: 180, paidFor: ['Zahin+Samiha', 'Zerin', 'Dipra'] },
    {
        description: 'Dinner at Bargee',
        paidBy: 'Zerin',
        amount: 980,
        paidFor: ['Zahin+Samiha', 'Zerin', 'Dipra'],
        items: [
            { description: 'Rice', amount: 140, paidFor: ['Zahin+Samiha', 'Zerin', 'Dipra'] },
            { description: 'Chicken', amount: 250, paidFor: ['Zahin+Samiha'] },
            { description: 'Vegetable', amount: 80, paidFor: ['Zahin+Samiha', 'Zerin', 'Dipra'] },
            { description: 'Dal', amount: 140, paidFor: ['Zahin+Samiha', 'Zerin', 'Dipra'] },
            { description: 'Alu vorta', amount: 100, paidFor: ['Zerin', 'Dipra'] },
            { description: 'Tomato vorta', amount: 50, paidFor: ['Zahin+Samiha', 'Zerin', 'Dipra'] },
            { description: 'Chapila fry', amount: 120, paidFor: ['Zahin+Samiha'] },
            { description: 'Water', amount: 40, paidFor: ['Zahin+Samiha', 'Zerin', 'Dipra'] },
            { description: 'Dew', amount: 30, paidFor: ['Zahin+Samiha'] },
            { description: 'Pepsi', amount: 30, paidFor: ['Dipra'] },
        ]
    }
];

const people = ref<Person[]>([]);
const expenses = ref<Expense[]>([]);
const newPerson = ref({ name: '', count: 1 });
const showRemoveConfirmation = ref(false);
const personToRemove = ref<{ idx: number, person: Person } | null>(null);
const selectedNewPayer = ref('');
const showClearConfirmation = ref(false);
const showImportError = ref(false);
const importErrorMessage = ref('');
const expenseForm = ref<Expense>({ 
  description: '', 
  paidBy: '', 
  amount: 0, 
  paidFor: [],
  items: []
});
const editingExpenseIdx = ref<number | null>(null);
const showExpenseForm = ref(false);
const showItemForm = ref(false);
const showPaidForCheckboxes = ref(false);
const currentItem = ref<ExpenseItem>({ description: '', amount: 0, paidFor: [] });
const expandedExpenses = ref<Set<number>>(new Set());

// IndexedDB setup
const dbName = 'tripExpenseTracker';
const dbVersion = 1;

function initDB(): Promise<IDBDatabase> {
    return new Promise((resolve, reject) => {
        const request = indexedDB.open(dbName, dbVersion);
        
        request.onerror = () => reject(request.error);
        request.onsuccess = () => resolve(request.result);
        
        request.onupgradeneeded = (event) => {
            const db = (event.target as IDBOpenDBRequest).result;
            if (!db.objectStoreNames.contains('trips')) {
                db.createObjectStore('trips');
            }
        };
    });
}

const currentTripId = ref<string>('');
const trips = ref<{ id: string; name: string }[]>([]);
const showNewTripForm = ref(false);
const newTripName = ref('');

function generateTripId() {
    return Date.now().toString(36) + Math.random().toString(36).substr(2);
}

function createNewTrip() {
    if (!newTripName.value.trim()) return;
    
    const id = generateTripId();
    const tripData: TripData = {
        id,
        name: newTripName.value.trim(),
        people: [],
        expenses: []
    };
    
    // Save the new trip
    saveTripToDB(tripData);
    
    // Update local state
    trips.value.push({ id, name: tripData.name });
    currentTripId.value = id;
    people.value = [];
    expenses.value = [];
    
    // Reset form
    newTripName.value = '';
    showNewTripForm.value = false;
}

async function saveTripToDB(tripData: TripData) {
    try {
        const db = await initDB();
        const tx = db.transaction(['trips'], 'readwrite');
        const dataStore = tx.objectStore('trips');
        
        await new Promise((resolve, reject) => {
            const request = dataStore.put(tripData, tripData.id);
            request.onsuccess = () => resolve(request.result);
            request.onerror = () => reject(request.error);
        });
        
        await new Promise((resolve, reject) => {
            tx.oncomplete = () => resolve(true);
            tx.onerror = () => reject(tx.error);
        });
    } catch (error) {
        console.error('Error saving trip to IndexedDB:', error);
    }
}

async function saveToDB() {
    if (!currentTripId.value) return;
    
    try {
        const tripData: TripData = {
            id: currentTripId.value,
            name: trips.value.find(t => t.id === currentTripId.value)?.name || '',
            people: people.value.map(person => ({
                name: person.name,
                count: person.count
            })),
            expenses: expenses.value.map(expense => ({
                description: expense.description,
                paidBy: expense.paidBy,
                amount: expense.amount,
                paidFor: [...expense.paidFor],
                items: expense.items ? expense.items.map(item => ({
                    description: item.description,
                    amount: item.amount,
                    paidFor: [...item.paidFor]
                })) : undefined
            }))
        };
        
        await saveTripToDB(tripData);
    } catch (error) {
        console.error('Error saving to IndexedDB:', error);
    }
}

async function loadFromDB() {
    try {
        const db = await initDB();
        const tx = db.transaction(['trips'], 'readonly');
        const dataStore = tx.objectStore('trips');
        
        // Load all trips
        const allTrips = await new Promise<TripData[]>((resolve, reject) => {
            const request = dataStore.getAll();
            request.onsuccess = () => resolve(request.result);
            request.onerror = () => reject(request.error);
        });
        
        // Update trips list
        trips.value = allTrips.map(trip => ({ id: trip.id, name: trip.name }));
        
        // If there's a current trip ID, load its data
        if (currentTripId.value) {
            const currentTrip = allTrips.find(trip => trip.id === currentTripId.value);
            if (currentTrip) {
                people.value = currentTrip.people;
                expenses.value = currentTrip.expenses;
            }
        }
    } catch (error) {
        console.error('Error loading from IndexedDB:', error);
    }
}

async function loadTrip(tripId: string) {
    try {
        const db = await initDB();
        const tx = db.transaction(['trips'], 'readonly');
        const dataStore = tx.objectStore('trips');
        
        const trip = await new Promise<TripData | undefined>((resolve, reject) => {
            const request = dataStore.get(tripId);
            request.onsuccess = () => resolve(request.result);
            request.onerror = () => reject(request.error);
        });
        
        if (trip) {
            currentTripId.value = trip.id;
            people.value = trip.people;
            expenses.value = trip.expenses;
        }
    } catch (error) {
        console.error('Error loading trip:', error);
    }
}

async function deleteTrip(tripId: string) {
    try {
        const db = await initDB();
        const tx = db.transaction(['trips'], 'readwrite');
        const dataStore = tx.objectStore('trips');
        
        await new Promise((resolve, reject) => {
            const request = dataStore.delete(tripId);
            request.onsuccess = () => resolve(request.result);
            request.onerror = () => reject(request.error);
        });
        
        // Update local state
        trips.value = trips.value.filter(t => t.id !== tripId);
        if (currentTripId.value === tripId) {
            currentTripId.value = '';
            people.value = [];
            expenses.value = [];
        }
    } catch (error) {
        console.error('Error deleting trip:', error);
    }
}

async function clearDB() {
    try {
        const db = await initDB();
        const tx = db.transaction(['trips'], 'readwrite');
        const dataStore = tx.objectStore('trips');
        
        await new Promise((resolve, reject) => {
            const request = dataStore.clear();
            request.onsuccess = () => resolve(true);
            request.onerror = () => reject(request.error);
        });
        
        await new Promise((resolve, reject) => {
            tx.oncomplete = () => resolve(true);
            tx.onerror = () => reject(tx.error);
        });
        
        trips.value = [];
        currentTripId.value = '';
        people.value = [];
        expenses.value = [];
        showClearConfirmation.value = false;
    } catch (error) {
        console.error('Error clearing IndexedDB:', error);
    }
}

// Load data when component mounts
onMounted(() => {
    loadFromDB();
});

// Save data when it changes
watch([people, expenses], () => {
    saveToDB();
}, { deep: true });

function addPerson() {
    const name = newPerson.value.name.trim();
    if (name && !people.value.some(p => p.name === name)) {
        people.value.push({ ...newPerson.value });
        newPerson.value = { name: '', count: 1 };
    }
}

function confirmRemovePerson(idx: number) {
    personToRemove.value = { idx, person: people.value[idx] };
    showRemoveConfirmation.value = true;
}

function handleRemovePerson() {
    if (!personToRemove.value) return;

    const { idx, person } = personToRemove.value;
    const personExpenses = expenses.value.filter(e => e.paidBy === person.name);

    if (personExpenses.length > 0 && selectedNewPayer.value) {
        // Reassign expenses to the selected person
        expenses.value = expenses.value.map(expense =>
            expense.paidBy === person.name ? { ...expense, paidBy: selectedNewPayer.value } : expense
        );
    } else if (personExpenses.length > 0) {
        // Delete expenses if no new payer selected
        expenses.value = expenses.value.filter(e => e.paidBy !== person.name);
    }

    // Remove person from paidFor arrays in all expenses
    expenses.value = expenses.value.map(expense => ({
        ...expense,
        paidFor: expense.paidFor.filter(p => p !== person.name)
    })).filter(expense => expense.paidFor.length > 0);

  people.value.splice(idx, 1);
    cancelRemoveConfirmation();
}

function cancelRemoveConfirmation() {
    showRemoveConfirmation.value = false;
    personToRemove.value = null;
    selectedNewPayer.value = '';
}

function toggleExpenseDetails(idx: number) {
    if (expandedExpenses.value.has(idx)) {
        expandedExpenses.value.delete(idx);
    } else {
        expandedExpenses.value.add(idx);
    }
}

function addItem() {
    if (!currentItem.value.description || !currentItem.value.amount || currentItem.value.paidFor.length === 0) return;
    expenseForm.value.items = expenseForm.value.items || [];
    expenseForm.value.items.push({ ...currentItem.value });
    currentItem.value = { description: '', amount: 0, paidFor: [] };
    showItemForm.value = false;
}

function removeItem(idx: number) {
    expenseForm.value.items?.splice(idx, 1);
}

function updateTotalAmount() {
    if (expenseForm.value.items?.length) {
        expenseForm.value.amount = expenseForm.value.items.reduce((sum, item) => sum + item.amount, 0);
    }
}

function openExpenseForm() {
    showExpenseForm.value = true;
    expenseForm.value = { 
        description: '', 
        paidBy: '', 
        amount: 0, 
        paidFor: [...people.value.map(p => p.name)], // Initialize with all people
        items: [] 
    };
    showPaidForCheckboxes.value = false;
    editingExpenseIdx.value = null;
}

function editExpense(idx: number) {
    expenseForm.value = { ...expenses.value[idx] };
    showPaidForCheckboxes.value = false;
    editingExpenseIdx.value = idx;
    showExpenseForm.value = true;
}

function saveExpense() {
  if (!expenseForm.value.description || !expenseForm.value.paidBy || !expenseForm.value.amount) return;
    if (expenseForm.value.items?.length) {
        // Validate that all items have valid data
        if (!expenseForm.value.items.every(item =>
            item.description && item.amount > 0 && item.paidFor.length > 0
        )) return;

        // Update the total amount from items
        updateTotalAmount();
    } else if (expenseForm.value.paidFor.length === 0) return;

  if (editingExpenseIdx.value === null) {
    expenses.value.push({ ...expenseForm.value });
  } else {
    expenses.value[editingExpenseIdx.value] = { ...expenseForm.value };
    editingExpenseIdx.value = null;
  }
    showExpenseForm.value = false;
    expenseForm.value = { description: '', paidBy: '', amount: 0, paidFor: [], items: [] };
}

function deleteExpense(idx: number) {
  expenses.value.splice(idx, 1);
  if (editingExpenseIdx.value === idx) {
    cancelEdit();
  }
}

function cancelEdit() {
  editingExpenseIdx.value = null;
    expenseForm.value = { description: '', paidBy: '', amount: 0, paidFor: [], items: [] };
    showExpenseForm.value = false;
}

// Settlement calculation
interface Settlement {
  from: string;
  to: string;
  amount: number;
}

const settlements = computed(() => {
  const balances: Record<string, number> = {}
  
    // First, add all payments to the payer's balance
  expenses.value.forEach(expense => {
    balances[expense.paidBy] = (balances[expense.paidBy] || 0) + expense.amount
  })

    // Then, subtract each person's share from their balance
    expenses.value.forEach(expense => {
        if (expense.items?.length) {
            // Handle itemized expenses
            expense.items.forEach(item => {
                const totalPeopleCount = item.paidFor.reduce((sum, name) => {
                    const person = people.value.find(p => p.name === name);
                    return sum + (person?.count || 1);
                }, 0);

                item.paidFor.forEach(name => {
                    const person = people.value.find(p => p.name === name);
                    const share = item.amount * (person?.count || 1) / totalPeopleCount;
                    balances[name] = (balances[name] || 0) - share;
                });
            });
        } else {
            // Handle regular expenses
            const totalPeopleCount = expense.paidFor.reduce((sum, name) => {
                const person = people.value.find(p => p.name === name);
                return sum + (person?.count || 1);
            }, 0);

            const perPersonShare = expense.amount / totalPeopleCount;

            expense.paidFor.forEach(name => {
                const person = people.value.find(p => p.name === name);
                const share = perPersonShare * (person?.count || 1);
                balances[name] = (balances[name] || 0) - share;
            });
        }
  })

  const settlements: Settlement[] = []
    const debtors = people.value.filter(p => (balances[p.name] || 0) < 0)
    const creditors = people.value.filter(p => (balances[p.name] || 0) > 0)

  debtors.forEach(debtor => {
        let debtAmount = Math.abs(balances[debtor.name])
    creditors.forEach(creditor => {
            if (debtAmount > 0 && balances[creditor.name] > 0) {
                const settlementAmount = Math.min(debtAmount, balances[creditor.name])
        if (settlementAmount > 0) {
          settlements.push({
                        from: debtor.name,
                        to: creditor.name,
            amount: Number(settlementAmount.toFixed(2))
          })
          debtAmount -= settlementAmount
                    balances[creditor.name] -= settlementAmount
        }
      }
    })
  })

  return settlements
})

const totalCost = computed(() => {
    return expenses.value.reduce((sum, expense) => sum + expense.amount, 0)
})

const perPersonCost = computed(() => {
    // Calculate total cost per person based on all expenses they're included in
    const personCosts: Record<string, number> = {};
    const totalPeopleCount = people.value.reduce((sum, person) => sum + person.count, 0);

    expenses.value.forEach(expense => {
        if (expense.items?.length) {
            // Handle itemized expenses
            expense.items.forEach(item => {
                const totalPeopleCount = item.paidFor.reduce((sum, name) => {
                    const person = people.value.find(p => p.name === name);
                    return sum + (person?.count || 1);
                }, 0);

                item.paidFor.forEach(name => {
                    const person = people.value.find(p => p.name === name);
                    const share = item.amount * (person?.count || 1) / totalPeopleCount;
                    personCosts[name] = (personCosts[name] || 0) + share;
                });
            });
        } else {
            // Handle regular expenses
            const totalPeopleCount = expense.paidFor.reduce((sum, name) => {
                const person = people.value.find(p => p.name === name);
                return sum + (person?.count || 1);
            }, 0);

            const perPersonShare = expense.amount / totalPeopleCount;

            expense.paidFor.forEach(name => {
                const person = people.value.find(p => p.name === name);
                const share = perPersonShare * (person?.count || 1);
                personCosts[name] = (personCosts[name] || 0) + share;
            });
        }
    });

    // Return the average cost per person
    const totalCost = Object.values(personCosts).reduce((sum, cost) => sum + cost, 0);
    return {
        average: Number((totalCost / totalPeopleCount).toFixed(2)),
        breakdown: personCosts
    };
});

function loadDefaultData() {
    people.value = [...defaultPeople];
    expenses.value = [...defaultExpenses];
}

function exportData() {
    const data = {
        people: people.value,
        expenses: expenses.value
    };
    const blob = new Blob([JSON.stringify(data, null, 2)], { type: 'application/json' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = `trip-expenses-${new Date().toISOString().split('T')[0]}.json`;
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    URL.revokeObjectURL(url);
}

function importData(event: Event) {
    const input = event.target as HTMLInputElement;
    if (!input.files?.length) return;

    const file = input.files[0];
    const reader = new FileReader();

    reader.onload = (e) => {
        try {
            const data = JSON.parse(e.target?.result as string);
            
            // Validate the data structure
            if (!Array.isArray(data.people) || !Array.isArray(data.expenses)) {
                throw new Error('Invalid data format');
            }

            // Validate people data
            if (!data.people.every((p: any) => p.name && typeof p.count === 'number')) {
                throw new Error('Invalid people data format');
            }

            // Validate expenses data
            if (!data.expenses.every((e: any) => 
                e.description && 
                e.paidBy && 
                typeof e.amount === 'number' && 
                Array.isArray(e.paidFor)
            )) {
                throw new Error('Invalid expenses data format');
            }

            // Clear current data and import new data
            people.value = data.people;
            expenses.value = data.expenses;
            
            // Reset the file input
            input.value = '';
        } catch (error) {
            showImportError.value = true;
            importErrorMessage.value = error instanceof Error ? error.message : 'Failed to import data';
            input.value = '';
        }
    };

    reader.readAsText(file);
}
</script>

<template>
  <div class="max-w-3xl mx-auto p-4">
    <div class="flex justify-between items-center mb-4">
      <h1 class="text-2xl font-bold">Trip Expense Tracker</h1>
      <div class="flex gap-2">
        <button 
          @click="showNewTripForm = true"
          class="bg-green-500 text-white px-4 py-2 rounded hover:bg-green-600 focus:ring-2 focus:ring-green-500 focus:ring-offset-2"
        >
          New Trip
        </button>
        <button 
          v-if="people.length === 0" 
          @click="loadDefaultData"
          class="bg-green-500 text-white px-4 py-2 rounded hover:bg-green-600 focus:ring-2 focus:ring-green-500 focus:ring-offset-2"
        >
          Load Sample Data from Kaptai Trip
        </button>
        <button 
          v-if="people.length > 0" 
          @click="showClearConfirmation = true"
          class="bg-red-500 text-white px-4 py-2 rounded hover:bg-red-600 focus:ring-2 focus:ring-red-500 focus:ring-offset-2"
        >
          Clear All Data
        </button>
      </div>
    </div>

    <!-- Trip Selection -->
    <div v-if="trips.length > 0" class="mb-6">
      <h2 class="font-semibold mb-2">Your Trips</h2>
      <div class="flex flex-wrap gap-2">
        <button 
          v-for="trip in trips" 
          :key="trip.id"
          @click="loadTrip(trip.id)"
          :class="[
            'px-4 py-2 rounded',
            currentTripId === trip.id 
              ? 'bg-blue-500 text-white' 
              : 'bg-gray-100 text-gray-700 hover:bg-gray-200'
          ]"
        >
          {{ trip.name }}
          <span 
            @click.stop="deleteTrip(trip.id)"
            class="ml-2 text-red-500 hover:text-red-700"
          >
            ×
          </span>
        </button>
      </div>
    </div>

    <!-- Remove Person Confirmation Dialog -->
    <div v-if="showRemoveConfirmation"
        class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center">
        <div class="bg-white p-6 rounded-lg max-w-md w-full">
            <h3 class="text-lg font-semibold mb-4">Remove {{ personToRemove?.person.name }}</h3>
            <p class="mb-4">This person has {{expenses.filter(e => e.paidBy === personToRemove?.person.name).length
                }} expenses. What would you like to do with them?</p>

            <div v-if="expenses.filter(e => e.paidBy === personToRemove?.person.name).length > 0" class="mb-4">
                <label class="block mb-2">Assign expenses to:</label>
                <select v-model="selectedNewPayer" class="w-full border rounded px-2 py-1">
                    <option value="">Delete expenses</option>
                    <option v-for="person in people.filter(p => p.name !== personToRemove?.person.name)"
                        :key="person.name" :value="person.name">
                        {{ person.name }}
                    </option>
                </select>
            </div>

            <div class="flex justify-end gap-2">
                <button @click="cancelRemoveConfirmation" class="px-4 py-2 text-gray-600 hover:text-gray-800">
                    Cancel
                </button>
                <button @click="handleRemovePerson"
                    class="px-4 py-2 bg-red-500 text-white rounded hover:bg-red-600">
                    Remove
                </button>
            </div>
        </div>
    </div>

    <!-- New Trip Form Modal -->
    <div v-if="showNewTripForm" class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center">
      <div class="bg-white p-6 rounded-lg max-w-md w-full">
        <h3 class="text-lg font-semibold mb-4">Create New Trip</h3>
        <form @submit.prevent="createNewTrip" class="space-y-4">
          <div>
            <label class="block text-sm font-medium text-gray-700 mb-1">Trip Name</label>
            <input 
              v-model="newTripName" 
              type="text" 
              placeholder="Enter trip name"
              class="w-full border rounded px-3 py-2 focus:ring-2 focus:ring-blue-500 focus:border-blue-500"
              required
            />
          </div>
          <div class="flex justify-end gap-2">
            <button 
              type="button" 
              @click="showNewTripForm = false"
              class="px-4 py-2 text-gray-700 bg-gray-100 rounded hover:bg-gray-200"
            >
              Cancel
            </button>
            <button 
              type="submit"
              class="px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600"
            >
              Create Trip
            </button>
          </div>
        </form>
      </div>
    </div>

    <!-- Import Error Modal -->
    <div v-if="showImportError" class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center">
      <div class="bg-white p-6 rounded-lg max-w-md w-full">
        <h3 class="text-lg font-semibold mb-4 text-red-600">Import Error</h3>
        <p class="mb-4">{{ importErrorMessage }}</p>
        <div class="flex justify-end">
          <button 
            @click="showImportError = false" 
            class="px-4 py-2 bg-gray-100 text-gray-700 rounded hover:bg-gray-200"
          >
            Close
          </button>
        </div>
      </div>
    </div>

    <!-- Clear Data Confirmation Modal -->
    <div v-if="showClearConfirmation" class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center">
      <div class="bg-white p-6 rounded-lg max-w-md w-full">
        <h3 class="text-lg font-semibold mb-4">Clear All Data</h3>
        <p class="mb-4">Are you sure you want to clear all data? This action cannot be undone.</p>
        <div class="flex justify-end gap-2">
          <button 
            @click="showClearConfirmation = false" 
            class="px-4 py-2 text-gray-700 bg-gray-100 rounded hover:bg-gray-200"
          >
            Cancel
          </button>
          <button 
            @click="clearDB" 
            class="px-4 py-2 bg-red-500 text-white rounded hover:bg-red-600"
          >
            Clear Data
          </button>
        </div>
      </div>
    </div>

    <div v-if="currentTripId">
      <!-- People List -->
      <div class="mb-6">
        <h2 class="font-semibold mb-2">People in the Trip</h2>
        <div class="flex flex-wrap gap-2 mb-2">
          <span v-for="(person, idx) in people" :key="person.name"
              class="bg-blue-100 text-blue-800 px-3 py-1 rounded-full text-sm">
              {{ person.name }}{{ person.count > 1 ? ` (${person.count})` : '' }}
              <button v-if="people.length > 1" @click="confirmRemovePerson(idx)"
                  class="ml-1 text-red-500 hover:text-red-700">&times;</button>
        </span>
        </div>
        <form @submit.prevent="addPerson" class="flex flex-wrap gap-2 items-end">
            <div>
                <label class="block text-sm font-medium text-gray-700 mb-1">Name</label>
                <input v-model="newPerson.name" type="text" placeholder="Add person"
                    class="border rounded px-2 py-1" />
            </div>
            <div>
                <label class="block text-sm font-medium text-gray-700 mb-1">Number of People</label>
                <input v-model.number="newPerson.count" type="number" min="1"
                    class="border rounded px-2 py-1 w-20" />
            </div>
    <button type="submit" class="bg-blue-500 text-white px-3 py-1 rounded hover:bg-blue-600">Add</button>
  </form>
      </div>

      <!-- Expenses List -->
      <div class="mb-6">
        <div class="flex justify-between items-center mb-2">
          <h2 class="font-semibold">Expenses</h2>
          <button @click="openExpenseForm"
              class="bg-blue-500 text-white px-3 py-1 rounded hover:bg-blue-600 focus:ring-2 focus:ring-blue-500 focus:ring-offset-2">
              Add Expense
          </button>
        </div>
        <table class="w-full text-left mb-2">
          <thead>
            <tr class="border-b">
              <th class="py-1">Description</th>
              <th class="py-1">Paid By</th>
              <th class="py-1">Amount</th>
              <th class="py-1">Actions</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(expense, idx) in expenses" :key="idx" class="border-b">
              <td>
                <div class="flex items-center gap-2">
                  <span>{{ expense.description }}</span>
                  <button 
                    v-if="expense.items?.length" 
                    @click="toggleExpenseDetails(idx)"
                    class="text-blue-500 hover:text-blue-700 text-sm"
                  >
                    {{ expandedExpenses.has(idx) ? 'Hide Details' : 'Show Details' }}
                  </button>
                </div>
                <div v-if="expense.items?.length && expandedExpenses.has(idx)" class="text-sm text-gray-600 mt-1">
                  <div v-for="(item, itemIdx) in expense.items" :key="itemIdx" class="flex justify-between">
                    <span>{{ item.description }}</span>
                    <span>৳{{ item.amount }}</span>
                  </div>
                  <div class="mt-2 pt-2 border-t border-gray-200">
                    <div class="font-medium text-gray-700">Per Person Breakdown:</div>
                    <div v-for="person in people" :key="person.name" class="flex justify-between">
                      <span>{{ person.name }}{{ person.count > 1 ? ` (${person.count})` : '' }}:</span>
                      <span>৳{{expense.items.reduce((sum, item) => {
                        if (!item.paidFor.includes(person.name)) return sum;
                        const totalPeopleCount = item.paidFor.reduce((total, name) => {
                          const p = people.find(p => p.name === name);
                          return total + (p?.count || 1);
                        }, 0);
                        return sum + (item.amount * (person.count / totalPeopleCount));
                      }, 0).toFixed(2)}}</span>
                    </div>
                  </div>
                </div>
              </td>
              <td>{{ expense.paidBy }}</td>
              <td>{{ expense.amount }}</td>
              <td>
                <button @click="editExpense(idx)" class="text-yellow-600 hover:underline mr-2">Edit</button>
                <button @click="deleteExpense(idx)" class="text-red-600 hover:underline">Delete</button>
              </td>
            </tr>
          </tbody>
        </table>
      </div>

      <!-- Expense Form Modal -->
      <div v-if="showExpenseForm" class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center">
        <div class="bg-white p-6 rounded-lg max-w-2xl w-full max-h-[90vh] overflow-y-auto">
          <div class="flex justify-between items-center mb-4">
            <h3 class="text-lg font-semibold">{{ editingExpenseIdx === null ? 'Add New Expense' : 'Edit Expense'
            }}</h3>
            <button @click="cancelEdit" class="text-gray-500 hover:text-gray-700">
              <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                  d="M6 18L18 6M6 6l12 12" />
              </svg>
            </button>
          </div>

          <form @submit.prevent="saveExpense" class="space-y-4">
            <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
              <div class="space-y-4">
                <div>
                  <label class="block text-sm font-medium text-gray-700 mb-1">Description</label>
                  <input v-model="expenseForm.description" type="text"
                    placeholder="What was the expense for?"
                    class="w-full border rounded px-3 py-2 focus:ring-2 focus:ring-blue-500 focus:border-blue-500"
                    required />
                </div>

                <div>
                  <label class="block text-sm font-medium text-gray-700 mb-1">Amount (৳)</label>
                  <input v-model.number="expenseForm.amount" type="number" min="1"
                    placeholder="Enter amount"
                    class="w-full border rounded px-3 py-2 focus:ring-2 focus:ring-blue-500 focus:border-blue-500"
                    :required="!expenseForm.items?.length" />
                </div>
              </div>

              <div class="space-y-4">
                <div>
                  <label class="block text-sm font-medium text-gray-700 mb-1">Paid By</label>
                  <select v-model="expenseForm.paidBy"
                    class="w-full border rounded px-3 py-2 focus:ring-2 focus:ring-blue-500 focus:border-blue-500"
                    required>
                    <option value="" disabled>Select who paid</option>
                    <option v-for="person in people" :key="person.name" :value="person.name">{{
                      person.name }}
                    </option>
                  </select>
                </div>

                <div v-if="!expenseForm.items?.length">
                  <label class="block text-sm font-medium text-gray-700 mb-1">Paid For</label>
                  <div class="bg-white border rounded p-3">
                    <div v-if="!showPaidForCheckboxes" class="flex justify-between items-center">
                      <span class="text-sm text-gray-600">
                        {{ expenseForm.paidFor.length === people.length ? 'All people' : `${expenseForm.paidFor.length} people selected` }}
                      </span>
                      <button 
                        type="button"
                        @click="showPaidForCheckboxes = true"
                        class="text-blue-500 hover:text-blue-700 text-sm"
                      >
                        Change
                      </button>
                    </div>
                    <div v-else class="grid grid-cols-2 gap-2">
                      <label v-for="person in people" :key="person.name"
                        class="flex items-center space-x-2">
                        <input type="checkbox" :value="person.name" v-model="expenseForm.paidFor"
                          class="rounded text-blue-500 focus:ring-blue-500">
                        <span class="text-sm">{{ person.name }}</span>
                      </label>
                    </div>
                  </div>
                </div>
              </div>
            </div>

            <!-- Itemized Expenses Section -->
            <div class="border-t pt-4">
              <div class="flex justify-between items-center mb-4">
                <h4 class="font-medium">Itemized Expenses</h4>
                <button type="button" @click="showItemForm = true"
                  class="text-blue-500 hover:text-blue-700">
                  + Add Item
                </button>
              </div>

              <div v-if="expenseForm.items?.length" class="space-y-2">
                <div v-for="(item, idx) in expenseForm.items" :key="idx"
                  class="flex justify-between items-center bg-gray-50 p-2 rounded">
                  <div>
                    <div class="font-medium">{{ item.description }}</div>
                    <div class="text-sm text-gray-600">
                      {{ item.paidFor.join(', ') }} - ৳{{ item.amount }}
                    </div>
                  </div>
                  <button type="button" @click="removeItem(idx)" class="text-red-500 hover:text-red-700">
                    Remove
                  </button>
                </div>
              </div>
            </div>

            <div class="flex justify-end space-x-3 pt-4 border-t">
              <button type="button" @click="cancelEdit"
                class="px-4 py-2 text-gray-700 bg-gray-100 rounded hover:bg-gray-200">
                Cancel
              </button>
              <button type="submit"
                class="px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600 focus:ring-2 focus:ring-blue-500 focus:ring-offset-2">
                {{ editingExpenseIdx === null ? 'Add Expense' : 'Update Expense' }}
              </button>
            </div>
          </form>
        </div>
      </div>

      <!-- Item Form Modal -->
      <div v-if="showItemForm" class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center">
        <div class="bg-white p-6 rounded-lg max-w-md w-full">
          <div class="flex justify-between items-center mb-4">
            <h3 class="text-lg font-semibold">Add Item</h3>
            <button @click="showItemForm = false" class="text-gray-500 hover:text-gray-700">
              <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                  d="M6 18L18 6M6 6l12 12" />
              </svg>
            </button>
          </div>

          <form @submit.prevent="addItem" class="space-y-4">
            <div>
              <label class="block text-sm font-medium text-gray-700 mb-1">Item Description</label>
              <input v-model="currentItem.description" type="text" placeholder="What was the item?"
                class="w-full border rounded px-3 py-2 focus:ring-2 focus:ring-blue-500 focus:border-blue-500"
                required />
            </div>

            <div>
              <label class="block text-sm font-medium text-gray-700 mb-1">Amount (৳)</label>
              <input v-model.number="currentItem.amount" type="number" min="1" placeholder="Enter amount"
                class="w-full border rounded px-3 py-2 focus:ring-2 focus:ring-blue-500 focus:border-blue-500"
                required />
            </div>

            <div>
              <label class="block text-sm font-medium text-gray-700 mb-1">Paid For</label>
              <div class="bg-white border rounded p-3">
                <div class="grid grid-cols-2 gap-2">
                  <label v-for="person in people" :key="person.name" class="flex items-center space-x-2">
                    <input type="checkbox" :value="person.name" v-model="currentItem.paidFor"
                      class="rounded text-blue-500 focus:ring-blue-500">
                    <span class="text-sm">{{ person.name }}</span>
                  </label>
                </div>
              </div>
            </div>

            <div class="flex justify-end space-x-3 pt-4 border-t">
              <button type="button" @click="showItemForm = false"
                class="px-4 py-2 text-gray-700 bg-gray-100 rounded hover:bg-gray-200">
                Cancel
              </button>
              <button type="submit"
                class="px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600 focus:ring-2 focus:ring-blue-500 focus:ring-offset-2">
                Add Item
              </button>
            </div>
          </form>
        </div>
      </div>

      <!-- Settlement Calculation -->
      <div>
        <h2 class="font-semibold mb-2">Settlement</h2>
        <div class="mb-4">
          <div class="text-lg">
            Total Cost: <span class="font-bold">৳{{ totalCost }}</span>
          </div>
          <div class="mt-2 text-sm">
            <div class="font-medium text-gray-700 mb-1">Individual Costs:</div>
            <div v-for="person in people" :key="person.name" class="flex justify-between">
              <span>{{ person.name }}{{ person.count > 1 ? ` (${person.count})` : '' }}:</span>
              <span>৳{{ perPersonCost.breakdown[person.name]?.toFixed(2) || '0.00' }}</span>
            </div>
          </div>
        </div>
        <div v-if="settlements.length === 0" class="text-gray-500">All settled up!</div>
        <ul>
          <li v-for="(s, idx) in settlements" :key="idx">
            <span class="font-mono">{{ s.from }}</span> owes <span class="font-mono">{{ s.to }}</span> <span
              class="font-semibold">৳{{ s.amount }}</span>
          </li>
        </ul>
      </div>
    </div>
    <div v-else class="text-center text-gray-500 py-8">
      Select a trip or create a new one to get started
    </div>
  </div>
</template>
