<script setup>
import { ref, reactive, computed, watch, nextTick, onUnmounted } from 'vue'
import { Chart, registerables } from 'chart.js'
import {
  onAuthStateChanged,
  signOut,
  createUserWithEmailAndPassword,
  signInWithEmailAndPassword
} from 'firebase/auth'
import {
  collection,
  addDoc,
  deleteDoc,
  doc,
  updateDoc,
  increment,
  onSnapshot
} from 'firebase/firestore'
import { auth, db } from './firebase'

Chart.register(...registerables)

const cats = ['Salary', 'Food', 'Rent', 'Transport', 'Entertainment', 'Health', 'Shopping', 'Utilities', 'Other']
const today = new Date().toISOString().slice(0, 10)
const palette = ['#5b3fe4', '#16a34a', '#f59e0b', '#ef4444', '#06b6d4', '#ec4899', '#84cc16', '#8b5cf6', '#64748b']
const fmt = n => n.toLocaleString(undefined, { style: 'currency', currency: 'USD', maximumFractionDigits: 2 })

// ---------- Auth ----------
const user = ref(null)
const ready = ref(false)
const err = ref('')
const a = reactive({ email: '', password: '', signup: false })
let unsubs = []

onAuthStateChanged(auth, u => {
  user.value = u
  ready.value = true
  unsubs.forEach(f => f())
  unsubs = []
  txs.value = []
  goals.value = []

  if (u) {
    const col = n => collection(db, 'users', u.uid, n)
    unsubs.push(onSnapshot(col('transactions'), s => {
      txs.value = s.docs.map(d => ({ id: d.id, ...d.data() }))
    }))
    unsubs.push(onSnapshot(col('goals'), s => {
      goals.value = s.docs.map(d => ({ id: d.id, add: null, ...d.data() }))
    }))
  }
})

onUnmounted(() => unsubs.forEach(f => f()))

async function submitAuth() {
  err.value = ''
  try {
    const fn = a.signup ? createUserWithEmailAndPassword : signInWithEmailAndPassword
    await fn(auth, a.email, a.password)
  } catch (e) {
    err.value = e.message.replace('Firebase: ', '')
  }
}

// ---------- Data ----------
const txs = ref([])
const goals = ref([])
const f = reactive({ type: 'expense', amount: null, cat: 'Food', date: today, note: '' })
const g = reactive({ name: '', target: null })
const fMonth = ref('all')
const fCat = ref('all')

const months = computed(() => [...new Set(txs.value.map(t => t.date.slice(0, 7)))].sort().reverse())

const shown = computed(() => txs.value
  .filter(t => (fMonth.value === 'all' || t.date.startsWith(fMonth.value)) && (fCat.value === 'all' || t.cat === fCat.value))
  .sort((x, y) => y.date.localeCompare(x.date)))

const totals = computed(() => ({
  inc: shown.value.filter(t => t.type === 'income').reduce((s, t) => s + t.amount, 0),
  exp: shown.value.filter(t => t.type === 'expense').reduce((s, t) => s + t.amount, 0)
}))

const txCol = () => collection(db, 'users', user.value.uid, 'transactions')
const goalCol = () => collection(db, 'users', user.value.uid, 'goals')

async function add() {
  await addDoc(txCol(), {
    type: f.type,
    amount: Number(f.amount),
    cat: f.cat,
    date: f.date,
    note: f.note
  })
  f.amount = null
  f.note = ''
}

const del = id => deleteDoc(doc(txCol(), id))

async function addGoal() {
  await addDoc(goalCol(), {
    name: g.name,
    target: Number(g.target),
    saved: 0
  })
  g.name = ''
  g.target = null
}

const contribute = async x => {
  if (x.add > 0) {
    await updateDoc(doc(goalCol(), x.id), { saved: increment(Number(x.add)) })
    x.add = null
  }
}

const delGoal = id => deleteDoc(doc(goalCol(), id))

// ---------- Charts ----------
const pie = ref(null)
const bar = ref(null)
let pc = null
let bc = null

function draw() {
  if (!pie.value || !bar.value) return

  const by = {}
  shown.value.filter(t => t.type === 'expense').forEach(t => {
    by[t.cat] = (by[t.cat] || 0) + t.amount
  })

  pc?.destroy()
  bc?.destroy()

  pc = new Chart(pie.value, {
    type: 'doughnut',
    data: {
      labels: Object.keys(by),
      datasets: [{ data: Object.values(by), backgroundColor: palette }]
    },
    options: {
      maintainAspectRatio: false,
      plugins: { legend: { position: 'bottom' } }
    }
  })

  const bm = {}
  txs.value.filter(t => fCat.value === 'all' || t.cat === fCat.value).forEach(t => {
    const k = t.date.slice(0, 7)
    bm[k] = bm[k] || { i: 0, e: 0 }
    t.type === 'income' ? (bm[k].i += t.amount) : (bm[k].e += t.amount)
  })

  const ks = Object.keys(bm).sort()
  bc = new Chart(bar.value, {
    type: 'bar',
    data: {
      labels: ks,
      datasets: [
        { label: 'Income', data: ks.map(k => bm[k].i), backgroundColor: '#16a34a' },
        { label: 'Expenses', data: ks.map(k => bm[k].e), backgroundColor: '#dc2626' }
      ]
    },
    options: {
      maintainAspectRatio: false,
      plugins: { legend: { position: 'bottom' } }
    }
  })
}

watch([txs, fMonth, fCat, user], () => nextTick(draw), { deep: true })
</script>

<template>
  <div v-if="!ready" class="wrap">
    <p class="sub">Loading…</p>
  </div>

  <div v-else-if="!user" class="card auth">
    <h1>Budget Planner</h1>
    <p class="sub">{{ a.signup ? 'Create an account' : 'Sign in to your budget' }}</p>
    <form @submit.prevent="submitAuth">
      <input v-model="a.email" type="email" placeholder="Email" required>
      <input v-model="a.password" type="password" minlength="6" placeholder="Password (6+ characters)" required>
      <button type="submit">{{ a.signup ? 'Sign up' : 'Sign in' }}</button>
    </form>
    <p v-if="err" class="err">{{ err }}</p>
    <p class="sub" style="margin:14px 0 0">
      {{ a.signup ? 'Have an account?' : 'New here?' }}
      <button type="button" class="link" @click="a.signup = !a.signup">
        {{ a.signup ? 'Sign in' : 'Create one' }}
      </button>
    </p>
  </div>

  <div v-else class="wrap">
    <div class="head">
      <div>
        <h1>Budget Planner</h1>
        <p class="sub">{{ user.email }}</p>
      </div>
      <button class="ghost" @click="signOut(auth)">Sign out</button>
    </div>

    <div class="grid stats">
      <div class="card stat"><span>Income</span><b class="inc">{{ fmt(totals.inc) }}</b></div>
      <div class="card stat"><span>Expenses</span><b class="exp">{{ fmt(totals.exp) }}</b></div>
      <div class="card stat"><span>Balance</span><b>{{ fmt(totals.inc - totals.exp) }}</b></div>
    </div>

    <div class="card" style="margin-bottom:14px">
      <h2>Add entry</h2>
      <form @submit.prevent="add">
        <select v-model="f.type">
          <option value="expense">Expense</option>
          <option value="income">Income</option>
        </select>
        <input v-model.number="f.amount" type="number" min="0" step="0.01" placeholder="Amount" required>
        <select v-model="f.cat">
          <option v-for="c in cats" :key="c">{{ c }}</option>
        </select>
        <input v-model="f.date" type="date" required>
        <input v-model="f.note" placeholder="Note (optional)">
        <button type="submit">Add</button>
      </form>
    </div>

    <div class="card" style="margin-bottom:14px">
      <div class="row">
        <select v-model="fMonth">
          <option value="all">All months</option>
          <option v-for="m in months" :key="m" :value="m">{{ m }}</option>
        </select>
        <select v-model="fCat">
          <option value="all">All categories</option>
          <option v-for="c in cats" :key="c">{{ c }}</option>
        </select>
      </div>
    </div>

    <div class="grid two">
      <div class="card">
        <h2>Spending by category</h2>
        <div class="chart"><canvas ref="pie"></canvas></div>
      </div>
      <div class="card">
        <h2>Income vs expenses by month</h2>
        <div class="chart"><canvas ref="bar"></canvas></div>
      </div>
    </div>

    <div class="grid two">
      <div class="card">
        <h2>Transactions</h2>
        <div v-if="!shown.length" class="empty">No entries match these filters.</div>
        <div v-else class="scroll">
          <table>
            <thead>
              <tr>
                <th>Date</th>
                <th>Category</th>
                <th>Note</th>
                <th>Amount</th>
                <th></th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="t in shown" :key="t.id">
                <td>{{ t.date }}</td>
                <td>{{ t.cat }}</td>
                <td>{{ t.note }}</td>
                <td :class="t.type === 'income' ? 'inc' : 'exp'">
                  {{ t.type === 'income' ? '+' : '-' }}{{ fmt(t.amount) }}
                </td>
                <td>
                  <button class="ghost" @click="del(t.id)" aria-label="Delete">X</button>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <div class="card">
        <h2>Savings goals</h2>
        <form @submit.prevent="addGoal" style="margin-bottom:6px">
          <input v-model="g.name" placeholder="Goal name" required>
          <input v-model.number="g.target" type="number" min="1" placeholder="Target" required>
          <button type="submit">Add goal</button>
        </form>
        <div v-if="!goals.length" class="empty">No goals yet.</div>
        <div v-for="x in goals" :key="x.id" class="goal">
          <div class="top">
            <strong>{{ x.name }}</strong>
            <span>{{ fmt(x.saved) }} / {{ fmt(x.target) }}</span>
          </div>
          <div class="bar">
            <i :style="{ width: Math.min(100, (x.saved / x.target) * 100) + '%' }"></i>
          </div>
          <div class="row">
            <input v-model.number="x.add" type="number" min="0" placeholder="Add savings" @keyup.enter="contribute(x)">
            <button @click="contribute(x)">Add</button>
            <button class="ghost" @click="delGoal(x.id)">Remove</button>
            <span style="color:var(--muted);align-self:center">
              {{ Math.min(100, Math.round((x.saved / x.target) * 100)) }}%
            </span>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>