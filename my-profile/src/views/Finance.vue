
<script setup>
import { ref, computed, onMounted } from 'vue'
import { supabase } from '../supabase'
import jsPDF from 'jspdf'
import autoTable from 'jspdf-autotable'

// =====================================================
// STATE
// =====================================================
const transactions = ref([])
const members = ref([])
const isLoading = ref(true)
const isSaving = ref(false)
const isModalOpen = ref(false)
const isEditing = ref(false)
const searchQuery = ref('')
const selectedYear = ref('')
const activeTab = ref('all')
const isDark = ref(false)
const errorMessage = ref('')

const categories = [
  'Pinjam',
  'Bayar Pinjaman',
  'Bayar Iuran',
  'Lahiran',
  'Meninggal Dunia',
  'Kecelakaan',
  'Nikahan',
  'Acara BMKG',
  'Lainnya'
]

const defaultForm = () => ({
  id: null,
  tanggal: getToday(),
  keterangan: '',
  kategori: 'Lainnya',
  tipe: 'Keluar',
  jumlah: '',
  anggota_id: '',
  pinjaman_id: null,
  status_pembayaran: null
})

const form = ref(defaultForm())

// =====================================================
// TANGGAL
// =====================================================
function getToday() {
  const now = new Date()
  const year = now.getFullYear()
  const month = String(now.getMonth() + 1).padStart(2, '0')
  const day = String(now.getDate()).padStart(2, '0')

  return `${year}-${month}-${day}`
}

function parseDate(dateString) {
  if (!dateString) return null

  const parts = String(dateString).slice(0, 10).split('-')
  if (parts.length !== 3) return null

  const [year, month, day] = parts.map(Number)

  if (!year || !month || !day) return null

  const date = new Date(year, month - 1, day)

  if (
    date.getFullYear() !== year ||
    date.getMonth() !== month - 1 ||
    date.getDate() !== day
  ) {
    return null
  }

  date.setHours(0, 0, 0, 0)
  return date
}

// Menambahkan 3 bulan kalender tanpa masalah overflow tanggal.
// Contoh: 31 Januari + 3 bulan = 30 April.
function addThreeMonths(dateString) {
  const date = parseDate(dateString)
  if (!date) return null

  const originalDay = date.getDate()
  const targetMonth = date.getMonth() + 3
  const targetYear = date.getFullYear() + Math.floor(targetMonth / 12)
  const normalizedMonth = targetMonth % 12

  const lastDay = new Date(
    targetYear,
    normalizedMonth + 1,
    0
  ).getDate()

  const result = new Date(
    targetYear,
    normalizedMonth,
    Math.min(originalDay, lastDay)
  )

  const year = result.getFullYear()
  const month = String(result.getMonth() + 1).padStart(2, '0')
  const day = String(result.getDate()).padStart(2, '0')

  return `${year}-${month}-${day}`
}

function formatDate(dateString) {
  if (!dateString) return '-'

  const date = parseDate(dateString)
  if (!date) return '-'

  return new Intl.DateTimeFormat('id-ID', {
    day: '2-digit',
    month: 'short',
    year: 'numeric'
  }).format(date)
}

function formatCurrency(value) {
  return new Intl.NumberFormat('id-ID', {
    style: 'currency',
    currency: 'IDR',
    maximumFractionDigits: 0
  }).format(Number(value) || 0)
}

// =====================================================
// DATA ANGGOTA
// =====================================================
async function fetchMembers() {
  const { data, error } = await supabase
    .from('anggota')
    .select('id, nama')
    .order('nama', { ascending: true })

  if (error) throw error

  members.value = data || []
}

function getMemberName(memberId) {
  if (!memberId) return '-'

  const member = members.value.find(
    item => String(item.id) === String(memberId)
  )

  return member?.nama || 'Anggota tidak ditemukan'
}

// =====================================================
// DATA KEUANGAN
// =====================================================
async function fetchTransactions() {
  const { data, error } = await supabase
    .from('keuangan')
    .select('*')
    .order('tanggal', { ascending: false })
    .order('id', { ascending: false })

  if (error) throw error

  transactions.value = data || []
}

async function refreshData() {
  isLoading.value = true
  errorMessage.value = ''

  try {
    await Promise.all([
      fetchMembers(),
      fetchTransactions()
    ])
  } catch (error) {
    console.error('Gagal mengambil data:', error)
    errorMessage.value =
      error.message || 'Gagal mengambil data dari database.'
  } finally {
    isLoading.value = false
  }
}

// =====================================================
// PINJAMAN DAN PEMBAYARAN
// =====================================================
function isLoan(item) {
  return item?.kategori === 'Pinjam'
}

function isLoanPayment(item) {
  return item?.kategori === 'Bayar Pinjaman'
}

function getLoanPayments(loanId, excludeTransactionId = null) {
  return transactions.value.filter(item =>
    isLoanPayment(item) &&
    String(item.pinjaman_id) === String(loanId) &&
    (
      excludeTransactionId == null ||
      String(item.id) !== String(excludeTransactionId)
    )
  )
}

function getPaidAmount(loanId, excludeTransactionId = null) {
  return getLoanPayments(loanId, excludeTransactionId)
    .reduce((total, item) => total + (Number(item.jumlah) || 0), 0)
}

function getRemaining(loanId, excludeTransactionId = null) {
  const loan = transactions.value.find(
    item => isLoan(item) && String(item.id) === String(loanId)
  )

  if (!loan) return 0

  return Math.max(
    0,
    (Number(loan.jumlah) || 0) -
    getPaidAmount(loanId, excludeTransactionId)
  )
}

function getDueDate(loan) {
  if (!loan || !isLoan(loan)) return null
  return addThreeMonths(loan.tanggal)
}

function getDueStatus(loan) {
  if (!loan || !isLoan(loan)) return '-'

  if (getRemaining(loan.id) <= 0) {
    return 'Lunas'
  }

  const dueDate = getDueDate(loan)
  if (!dueDate) return 'Tanggal tidak valid'

  // Lewat atau tepat pada tanggal jatuh tempo.
  return getToday() >= dueDate
    ? 'Jatuh Tempo'
    : 'Belum Jatuh Tempo'
}

function getDueClass(status) {
  if (status === 'Lunas') {
    return 'bg-green-100 text-green-700 dark:bg-green-900/40 dark:text-green-300'
  }

  if (status === 'Jatuh Tempo') {
    return 'bg-red-100 text-red-700 dark:bg-red-900/40 dark:text-red-300'
  }

  return 'bg-yellow-100 text-yellow-700 dark:bg-yellow-900/40 dark:text-yellow-300'
}

function getLoanOptions(excludeTransactionId = null) {
  return transactions.value
    .filter(item =>
      isLoan(item) &&
      String(item.id) !== String(excludeTransactionId) &&
      (
        getRemaining(item.id, excludeTransactionId) > 0 ||
        String(item.id) === String(form.value.pinjaman_id)
      )
    )
    .sort((a, b) => String(b.tanggal).localeCompare(String(a.tanggal)))
}

function onCategoryChange() {
  if (form.value.kategori === 'Pinjam') {
    form.value.tipe = 'Keluar'
    form.value.pinjaman_id = null
    form.value.status_pembayaran = 'Belum Lunas'
  } else if (form.value.kategori === 'Bayar Pinjaman') {
    form.value.tipe = 'Masuk'
    form.value.status_pembayaran = null
  } else {
    form.value.pinjaman_id = null
    form.value.anggota_id = ''
    form.value.status_pembayaran = null
  }
}

function onLoanChange() {
  const loan = transactions.value.find(
    item =>
      isLoan(item) &&
      String(item.id) === String(form.value.pinjaman_id)
  )

  if (loan) {
    form.value.anggota_id = loan.anggota_id || ''
  } else {
    form.value.anggota_id = ''
  }
}

// Perbarui status pembayaran pinjaman di database.
async function recalculateLoanStatus(loanId) {
  if (loanId == null) return

  const loan = transactions.value.find(
    item => isLoan(item) && String(item.id) === String(loanId)
  )

  if (!loan) return

  const paid = getPaidAmount(loanId)
  const remaining = Math.max(0, Number(loan.jumlah) - paid)

  const status = remaining <= 0 ? 'Lunas' : 'Belum Lunas'

  const { error } = await supabase
    .from('keuangan')
    .update({ status_pembayaran: status })
    .eq('id', loanId)

  if (error) throw error

  loan.status_pembayaran = status
}

// =====================================================
// SALDO DAN RINGKASAN
// =====================================================
const totalMasuk = computed(() =>
  transactions.value
    .filter(item => item.tipe === 'Masuk')
    .reduce((sum, item) => sum + (Number(item.jumlah) || 0), 0)
)

const totalKeluar = computed(() =>
  transactions.value
    .filter(item => item.tipe === 'Keluar')
    .reduce((sum, item) => sum + (Number(item.jumlah) || 0), 0)
)

const saldo = computed(() => totalMasuk.value - totalKeluar.value)

const totalPinjamanAktif = computed(() =>
  transactions.value
    .filter(item => isLoan(item))
    .reduce((sum, loan) => sum + getRemaining(loan.id), 0)
)

const years = computed(() => {
  const yearSet = new Set(
    transactions.value
      .map(item => item.tanggal?.slice(0, 4))
      .filter(Boolean)
  )

  return [...yearSet].sort((a, b) => b.localeCompare(a))
})

const filteredTransactions = computed(() => {
  const query = searchQuery.value.trim().toLowerCase()

  return transactions.value.filter(item => {
    const matchesSearch =
      !query ||
      String(item.keterangan || '').toLowerCase().includes(query) ||
      String(item.kategori || '').toLowerCase().includes(query) ||
      getMemberName(item.anggota_id).toLowerCase().includes(query)

    const matchesYear =
      !selectedYear.value ||
      item.tanggal?.slice(0, 4) === selectedYear.value

    const matchesTab =
      activeTab.value === 'all' ||
      (activeTab.value === 'loan' && isLoan(item)) ||
      (activeTab.value === 'payment' && isLoanPayment(item))

    return matchesSearch && matchesYear && matchesTab
  })
})

// =====================================================
// MODAL TAMBAH DAN EDIT
// =====================================================
function openAddModal() {
  isEditing.value = false
  form.value = defaultForm()
  errorMessage.value = ''
  isModalOpen.value = true
}

function openEditModal(item) {
  isEditing.value = true

  form.value = {
    ...defaultForm(),
    ...item,
    jumlah: Number(item.jumlah) || '',
    anggota_id: item.anggota_id || '',
    pinjaman_id: item.pinjaman_id ?? null
  }

  errorMessage.value = ''
  isModalOpen.value = true
}

function closeModal() {
  if (isSaving.value) return

  isModalOpen.value = false
  form.value = defaultForm()
}

// =====================================================
// SIMPAN TRANSAKSI
// =====================================================
async function handleSubmit() {
  errorMessage.value = ''

  const kategori = form.value.kategori
  const isNewLoan = kategori === 'Pinjam'
  const isPayment = kategori === 'Bayar Pinjaman'

  const jumlah = Number(form.value.jumlah)
  const tanggal = form.value.tanggal
  const keterangan = form.value.keterangan.trim()

  if (!tanggal || !parseDate(tanggal)) {
    errorMessage.value = 'Tanggal transaksi tidak valid.'
    return
  }

  if (!keterangan) {
    errorMessage.value = 'Keterangan wajib diisi.'
    return
  }

  if (!Number.isFinite(jumlah) || jumlah <= 0) {
    errorMessage.value = 'Jumlah harus lebih besar dari nol.'
    return
  }

  if (isNewLoan && !form.value.anggota_id) {
    errorMessage.value = 'Pilih anggota yang melakukan pinjaman.'
    return
  }

  if (
    form.value.anggota_id &&
    !members.value.some(
      member => String(member.id) === String(form.value.anggota_id)
    )
  ) {
    errorMessage.value = 'Anggota yang dipilih tidak ditemukan.'
    return
  }

  let selectedLoan = null

  if (isPayment) {
    if (!form.value.pinjaman_id) {
      errorMessage.value = 'Pilih pinjaman yang akan dibayar.'
      return
    }

    selectedLoan = transactions.value.find(
      item =>
        isLoan(item) &&
        String(item.id) === String(form.value.pinjaman_id)
    )

    if (!selectedLoan) {
      errorMessage.value = 'Data pinjaman tidak ditemukan.'
      return
    }

    const remaining = getRemaining(
      selectedLoan.id,
      isEditing.value && isLoanPayment(form.value)
        ? form.value.id
        : null
    )

    if (jumlah > remaining) {
      errorMessage.value =
        `Pembayaran melebihi sisa pinjaman ${formatCurrency(remaining)}.`
      return
    }
  }

  // Simpan tipe sesuai aturan kategori.
  const tipe = isNewLoan
    ? 'Keluar'
    : isPayment
      ? 'Masuk'
      : form.value.tipe

  const payload = {
    tanggal,
    keterangan,
    kategori,
    tipe,
    jumlah,
    anggota_id: isNewLoan
      ? form.value.anggota_id
      : isPayment
        ? selectedLoan.anggota_id || null
        : null,
    pinjaman_id: isPayment ? selectedLoan.id : null,
    status_pembayaran: isNewLoan ? 'Belum Lunas' : null
  }

  isSaving.value = true

  const oldTransaction = isEditing.value
    ? transactions.value.find(
        item => String(item.id) === String(form.value.id)
      )
    : null

  const affectedLoanIds = new Set()

  if (oldTransaction && isLoanPayment(oldTransaction)) {
    if (oldTransaction.pinjaman_id != null) {
      affectedLoanIds.add(String(oldTransaction.pinjaman_id))
    }
  }

  if (isPayment && selectedLoan?.id != null) {
    affectedLoanIds.add(String(selectedLoan.id))
  }

  try {
    let savedData

    if (isEditing.value) {
      const { data, error } = await supabase
        .from('keuangan')
        .update(payload)
        .eq('id', form.value.id)
        .select()
        .single()

      if (error) throw error
      savedData = data
    } else {
      const { data, error } = await supabase
        .from('keuangan')
        .insert(payload)
        .select()
        .single()

      if (error) throw error
      savedData = data
    }

    // Muat ulang agar perhitungan menggunakan data terbaru.
    await fetchTransactions()

    // Jika transaksi yang diedit adalah pinjaman, hitung ulang
    // status pinjaman tersebut juga.
    if (oldTransaction && isLoan(oldTransaction)) {
      affectedLoanIds.add(String(oldTransaction.id))
    }

    if (isNewLoan && savedData?.id != null) {
      affectedLoanIds.add(String(savedData.id))
    }

    for (const loanId of affectedLoanIds) {
      await recalculateLoanStatus(loanId)
    }

    await fetchTransactions()

    isModalOpen.value = false
    form.value = defaultForm()
  } catch (error) {
    console.error('Gagal menyimpan transaksi:', error)
    errorMessage.value =
      error.message || 'Terjadi kesalahan saat menyimpan data.'
  } finally {
    isSaving.value = false
  }
}

// =====================================================
// HAPUS TRANSAKSI
// =====================================================
async function deleteTransaction(item) {
  const isLoanItem = isLoan(item)
  const isPayment = isLoanPayment(item)

  if (isLoanItem && getPaidAmount(item.id) > 0) {
    alert(
      'Pinjaman ini sudah memiliki pembayaran. ' +
      'Hapus transaksi pembayarannya terlebih dahulu.'
    )
    return
  }

  const message = isPayment
    ? 'Yakin ingin menghapus transaksi pembayaran ini?'
    : 'Yakin ingin menghapus transaksi ini?'

  if (!confirm(message)) return

  errorMessage.value = ''

  try {
    const affectedLoanId = isPayment ? item.pinjaman_id : null

    const { error } = await supabase
      .from('keuangan')
      .delete()
      .eq('id', item.id)

    if (error) throw error

    await fetchTransactions()

    if (affectedLoanId != null) {
      await recalculateLoanStatus(affectedLoanId)
      await fetchTransactions()
    }
  } catch (error) {
    console.error('Gagal menghapus transaksi:', error)
    errorMessage.value =
      error.message || 'Gagal menghapus transaksi.'
  }
}

// =====================================================
// EKSPOR PDF
// =====================================================
function exportPDF() {
  try {
    const doc = new jsPDF({
      orientation: 'landscape',
      unit: 'mm',
      format: 'a4'
    })

    doc.setFontSize(16)
    doc.text('LAPORAN KEUANGAN BMKG', 14, 15)

    doc.setFontSize(9)
    doc.text(
      `Tanggal cetak: ${formatDate(getToday())}`,
      14,
      22
    )

    const rows = filteredTransactions.value.map(item => [
      formatDate(item.tanggal),
      getMemberName(item.anggota_id),
      item.keterangan || '-',
      item.kategori || '-',
      item.tipe || '-',
      formatCurrency(item.jumlah),
      isLoan(item)
        ? getDueStatus(item)
        : isLoanPayment(item)
          ? 'Pembayaran'
          : '-'
    ])

    autoTable(doc, {
      startY: 28,
      head: [[
        'Tanggal',
        'Nama Anggota',
        'Keterangan',
        'Kategori',
        'Tipe',
        'Jumlah',
        'Status Pinjaman'
      ]],
      body: rows,
      styles: {
        fontSize: 8,
        cellPadding: 2
      },
      headStyles: {
        fillColor: [31, 78, 121]
      }
    })

    const finalY = doc.lastAutoTable?.finalY || 35

    doc.setFontSize(10)
    doc.text(
      `Total Masuk: ${formatCurrency(totalMasuk.value)}`,
      14,
      finalY + 10
    )
    doc.text(
      `Total Keluar: ${formatCurrency(totalKeluar.value)}`,
      14,
      finalY + 17
    )
    doc.text(
      `Saldo: ${formatCurrency(saldo.value)}`,
      14,
      finalY + 24
    )

    doc.save(`Laporan-Keuangan-BMKG-${getToday()}.pdf`)
  } catch (error) {
    console.error('Gagal membuat PDF:', error)
    alert('Gagal membuat PDF. Periksa kembali data transaksi.')
  }
}

// =====================================================
// INISIALISASI
// =====================================================
onMounted(refreshData)
</script>

<template>
  <div
    class="min-h-screen p-4 sm:p-6 transition-colors"
    :class="isDark
      ? 'bg-slate-950 text-slate-100'
      : 'bg-slate-50 text-slate-800'"
  >
    <div class="mx-auto max-w-7xl space-y-6">
      <!-- HEADER -->
      <header class="flex flex-col gap-4 sm:flex-row sm:items-center sm:justify-between">
        <div>
          <h1 class="text-2xl font-bold sm:text-3xl">
            Keuangan BMKG
          </h1>
          <p
            class="mt-1 text-sm"
            :class="isDark ? 'text-slate-400' : 'text-slate-500'"
          >
            Kelola transaksi, pinjaman anggota, dan pembayaran.
          </p>
        </div>

        <div class="flex flex-wrap gap-2">
          <button
            type="button"
            class="rounded-lg border px-4 py-2 text-sm font-medium"
            :class="isDark
              ? 'border-slate-700 bg-slate-900 hover:bg-slate-800'
              : 'border-slate-200 bg-white hover:bg-slate-100'"
            @click="isDark = !isDark"
          >
            {{ isDark ? 'Mode Terang' : 'Mode Gelap' }}
          </button>

          <button
            type="button"
            class="rounded-lg bg-blue-600 px-4 py-2 text-sm font-semibold text-white hover:bg-blue-700"
            @click="exportPDF"
          >
            Ekspor PDF
          </button>

          <button
            type="button"
            class="rounded-lg bg-emerald-600 px-4 py-2 text-sm font-semibold text-white hover:bg-emerald-700"
            @click="openAddModal"
          >
            + Tambah Data
          </button>
        </div>
      </header>

      <!-- PESAN ERROR -->
      <div
        v-if="errorMessage"
        class="flex items-start justify-between gap-3 rounded-xl border border-red-200 bg-red-50 p-4 text-sm text-red-700 dark:border-red-900 dark:bg-red-950/50 dark:text-red-300"
      >
        <span>{{ errorMessage }}</span>
        <button
          type="button"
          class="font-bold"
          @click="errorMessage = ''"
        >
          ✕
        </button>
      </div>

      <!-- RINGKASAN -->
      <section class="grid grid-cols-1 gap-4 sm:grid-cols-2 xl:grid-cols-4">
        <div
          class="rounded-2xl border p-5 shadow-sm"
          :class="isDark ? 'border-slate-800 bg-slate-900' : 'border-slate-200 bg-white'"
        >
          <p class="text-sm text-slate-500 dark:text-slate-400">Total Masuk</p>
          <p class="mt-2 text-xl font-bold text-emerald-600">
            {{ formatCurrency(totalMasuk) }}
          </p>
        </div>

        <div
          class="rounded-2xl border p-5 shadow-sm"
          :class="isDark ? 'border-slate-800 bg-slate-900' : 'border-slate-200 bg-white'"
        >
          <p class="text-sm text-slate-500 dark:text-slate-400">Total Keluar</p>
          <p class="mt-2 text-xl font-bold text-red-600">
            {{ formatCurrency(totalKeluar) }}
          </p>
        </div>

        <div
          class="rounded-2xl border p-5 shadow-sm"
          :class="isDark ? 'border-slate-800 bg-slate-900' : 'border-slate-200 bg-white'"
        >
          <p class="text-sm text-slate-500 dark:text-slate-400">Saldo Kas</p>
          <p
            class="mt-2 text-xl font-bold"
            :class="saldo < 0 ? 'text-red-600' : 'text-blue-600'"
          >
            {{ formatCurrency(saldo) }}
          </p>
        </div>

        <div
          class="rounded-2xl border p-5 shadow-sm"
          :class="isDark ? 'border-slate-800 bg-slate-900' : 'border-slate-200 bg-white'"
        >
          <p class="text-sm text-slate-500 dark:text-slate-400">
            Sisa Pinjaman Anggota
          </p>
          <p class="mt-2 text-xl font-bold text-orange-600">
            {{ formatCurrency(totalPinjamanAktif) }}
          </p>
        </div>
      </section>

      <!-- FILTER -->
      <section
        class="rounded-2xl border p-4 shadow-sm sm:p-5"
        :class="isDark ? 'border-slate-800 bg-slate-900' : 'border-slate-200 bg-white'"
      >
        <div class="grid grid-cols-1 gap-3 md:grid-cols-3">
          <input
            v-model="searchQuery"
            type="search"
            placeholder="Cari keterangan atau nama anggota..."
            class="w-full rounded-lg border px-3 py-2.5 outline-none focus:border-blue-500 focus:ring-2 focus:ring-blue-100 dark:border-slate-700 dark:bg-slate-950"
          />

          <select
            v-model="selectedYear"
            class="w-full rounded-lg border px-3 py-2.5 outline-none focus:border-blue-500 dark:border-slate-700 dark:bg-slate-950"
          >
            <option value="">Semua Tahun</option>
            <option
              v-for="year in years"
              :key="year"
              :value="year"
            >
              {{ year }}
            </option>
          </select>

          <button
            type="button"
            class="rounded-lg border px-3 py-2.5 text-sm font-medium dark:border-slate-700"
            @click="refreshData"
          >
            Muat Ulang Data
          </button>
        </div>

        <div class="mt-4 flex flex-wrap gap-2">
          <button
            v-for="tab in [
              { value: 'all', label: 'Semua Transaksi' },
              { value: 'loan', label: 'Pinjaman' },
              { value: 'payment', label: 'Pembayaran Pinjaman' }
            ]"
            :key="tab.value"
            type="button"
            class="rounded-full px-4 py-2 text-sm font-medium"
            :class="activeTab === tab.value
              ? 'bg-blue-600 text-white'
              : isDark
                ? 'bg-slate-800 text-slate-300'
                : 'bg-slate-100 text-slate-600'"
            @click="activeTab = tab.value"
          >
            {{ tab.label }}
          </button>
        </div>
      </section>

      <!-- TABEL TRANSAKSI -->
      <section
        class="overflow-hidden rounded-2xl border shadow-sm"
        :class="isDark ? 'border-slate-800 bg-slate-900' : 'border-slate-200 bg-white'"
      >
        <div class="flex items-center justify-between gap-3 p-5">
          <div>
            <h2 class="text-lg font-bold">Daftar Transaksi</h2>
            <p class="mt-1 text-sm text-slate-500 dark:text-slate-400">
              {{ filteredTransactions.length }} transaksi ditampilkan
            </p>
          </div>
        </div>

        <div v-if="isLoading" class="p-10 text-center text-slate-500">
          Memuat data...
        </div>

        <div
          v-else-if="filteredTransactions.length === 0"
          class="p-10 text-center text-slate-500"
        >
          Tidak ada data transaksi.
        </div>

        <div v-else class="overflow-x-auto">
          <table class="w-full min-w-[1100px] text-left text-sm">
            <thead
              :class="isDark
                ? 'bg-slate-800 text-slate-300'
                : 'bg-slate-100 text-slate-600'"
            >
              <tr>
                <th class="px-4 py-3 font-semibold">Tanggal</th>
                <th class="px-4 py-3 font-semibold">Nama Anggota</th>
                <th class="px-4 py-3 font-semibold">Keterangan</th>
                <th class="px-4 py-3 font-semibold">Kategori</th>
                <th class="px-4 py-3 font-semibold">Tipe</th>
                <th class="px-4 py-3 text-right font-semibold">Jumlah</th>
                <th class="px-4 py-3 font-semibold">Status Pinjaman</th>
                <th class="px-4 py-3 text-center font-semibold">Aksi</th>
              </tr>
            </thead>

            <tbody>
              <tr
                v-for="item in filteredTransactions"
                :key="item.id"
                class="border-t dark:border-slate-800"
                :class="isDark ? 'hover:bg-slate-800/50' : 'hover:bg-slate-50'"
              >
                <td class="whitespace-nowrap px-4 py-3">
                  {{ formatDate(item.tanggal) }}
                </td>

                <td class="px-4 py-3 font-medium">
                  {{ getMemberName(item.anggota_id) }}
                </td>

                <td class="max-w-xs px-4 py-3">
                  <span class="line-clamp-2">
                    {{ item.keterangan || '-' }}
                  </span>
                </td>

                <td class="px-4 py-3">
                  {{ item.kategori }}
                </td>

                <td class="px-4 py-3">
                  <span
                    class="rounded-full px-2.5 py-1 text-xs font-semibold"
                    :class="item.tipe === 'Masuk'
                      ? 'bg-emerald-100 text-emerald-700'
                      : 'bg-red-100 text-red-700'"
                  >
                    {{ item.tipe }}
                  </span>
                </td>

                <td class="whitespace-nowrap px-4 py-3 text-right font-semibold">
                  {{ formatCurrency(item.jumlah) }}
                </td>

                <td class="px-4 py-3">
                  <span
                    v-if="isLoan(item)"
                    class="whitespace-nowrap rounded-full px-2.5 py-1 text-xs font-semibold"
                    :class="getDueClass(getDueStatus(item))"
                  >
                    {{ getDueStatus(item) }}
                  </span>

                  <span
                    v-else-if="isLoanPayment(item)"
                    class="text-xs text-slate-500"
                  >
                    Pembayaran
                  </span>

                  <span v-else class="text-slate-400">-</span>
                </td>

                <td class="px-4 py-3">
                  <div class="flex justify-center gap-2">
                    <button
                      type="button"
                      class="rounded-md bg-amber-100 px-3 py-1.5 text-xs font-semibold text-amber-700 hover:bg-amber-200"
                      @click="openEditModal(item)"
                    >
                      Edit
                    </button>

                    <button
                      type="button"
                      class="rounded-md bg-red-100 px-3 py-1.5 text-xs font-semibold text-red-700 hover:bg-red-200"
                      @click="deleteTransaction(item)"
                    >
                      Hapus
                    </button>
                  </div>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </section>
    </div>

    <!-- MODAL TAMBAH / EDIT -->
    <div
      v-if="isModalOpen"
      class="fixed inset-0 z-50 flex items-center justify-center overflow-y-auto bg-black/50 p-4"
      @click.self="closeModal"
    >
      <section
        class="my-4 max-h-[92vh] w-full max-w-2xl overflow-y-auto rounded-2xl p-6 shadow-2xl"
        :class="isDark ? 'bg-slate-900 text-slate-100' : 'bg-white text-slate-800'"
      >
        <div class="mb-6 flex items-center justify-between gap-3">
          <div>
            <h2 class="text-xl font-bold">
              {{ isEditing ? 'Edit Transaksi' : 'Tambah Transaksi' }}
            </h2>
            <p class="mt-1 text-sm text-slate-500 dark:text-slate-400">
              Isi data transaksi dengan benar.
            </p>
          </div>

          <button
            type="button"
            class="rounded-lg px-3 py-2 text-xl hover:bg-slate-100 dark:hover:bg-slate-800"
            :disabled="isSaving"
            @click="closeModal"
          >
            ✕
          </button>
        </div>

        <form class="space-y-4" @submit.prevent="handleSubmit">
          <!-- TANGGAL -->
          <div>
            <label class="mb-1.5 block text-sm font-semibold">
              Tanggal Transaksi
            </label>
            <input
              v-model="form.tanggal"
              type="date"
              required
              class="w-full rounded-lg border px-3 py-2.5 outline-none focus:border-blue-500 dark:border-slate-700 dark:bg-slate-950"
            />
          </div>

          <!-- KATEGORI -->
          <div>
            <label class="mb-1.5 block text-sm font-semibold">
              Kategori
            </label>
            <select
              v-model="form.kategori"
              required
              class="w-full rounded-lg border px-3 py-2.5 outline-none focus:border-blue-500 dark:border-slate-700 dark:bg-slate-950"
              @change="onCategoryChange"
            >
              <option
                v-for="category in categories"
                :key="category"
                :value="category"
              >
                {{ category }}
              </option>
            </select>
          </div>

          <!-- NAMA ANGGOTA UNTUK PINJAMAN -->
          <div v-if="form.kategori === 'Pinjam'">
            <label class="mb-1.5 block text-sm font-semibold">
              Nama Anggota
            </label>
            <select
              v-model="form.anggota_id"
              required
              class="w-full rounded-lg border px-3 py-2.5 outline-none focus:border-blue-500 dark:border-slate-700 dark:bg-slate-950"
            >
              <option value="" disabled>
                Pilih anggota yang meminjam
              </option>
              <option
                v-for="member in members"
                :key="member.id"
                :value="member.id"
              >
                {{ member.nama }}
              </option>
            </select>

            <p
              v-if="members.length === 0"
              class="mt-1 text-sm text-red-600"
            >
              Belum ada anggota. Tambahkan anggota terlebih dahulu.
            </p>
          </div>

          <!-- PILIH PINJAMAN UNTUK PEMBAYARAN -->
          <div v-if="form.kategori === 'Bayar Pinjaman'">
            <label class="mb-1.5 block text-sm font-semibold">
              Pinjaman yang Dibayar
            </label>
            <select
              v-model="form.pinjaman_id"
              required
              class="w-full rounded-lg border px-3 py-2.5 outline-none focus:border-blue-500 dark:border-slate-700 dark:bg-slate-950"
              @change="onLoanChange"
            >
              <option :value="null" disabled>
                Pilih pinjaman
              </option>

              <option
                v-for="loan in getLoanOptions(
                  isEditing && isLoanPayment(form) ? form.id : null
                )"
                :key="loan.id"
                :value="loan.id"
              >
                {{ getMemberName(loan.anggota_id) }}
                — {{ loan.keterangan }}
                — Sisa {{ formatCurrency(getRemaining(
                  loan.id,
                  isEditing && isLoanPayment(form) ? form.id : null
                )) }}
              </option>
            </select>

            <p
              v-if="getLoanOptions(
                isEditing && isLoanPayment(form) ? form.id : null
              ).length === 0"
              class="mt-1 text-sm text-amber-600"
            >
              Tidak ada pinjaman dengan sisa pembayaran.
            </p>

            <div
              v-if="form.pinjaman_id"
              class="mt-2 rounded-lg bg-blue-50 p-3 text-sm text-blue-800 dark:bg-blue-950 dark:text-blue-200"
            >
              Anggota:
              <strong>{{ getMemberName(form.anggota_id) }}</strong>
            </div>
          </div>

          <!-- KETERANGAN -->
          <div>
            <label class="mb-1.5 block text-sm font-semibold">
              Keterangan
            </label>
            <textarea
              v-model="form.keterangan"
              rows="3"
              required
              placeholder="Masukkan keterangan transaksi"
              class="w-full rounded-lg border px-3 py-2.5 outline-none focus:border-blue-500 dark:border-slate-700 dark:bg-slate-950"
            />
          </div>

          <!-- TIPE -->
          <div v-if="!['Pinjam', 'Bayar Pinjaman'].includes(form.kategori)">
            <label class="mb-1.5 block text-sm font-semibold">
              Tipe Transaksi
            </label>
            <select
              v-model="form.tipe"
              required
              class="w-full rounded-lg border px-3 py-2.5 outline-none focus:border-blue-500 dark:border-slate-700 dark:bg-slate-950"
            >
              <option value="Masuk">Masuk</option>
              <option value="Keluar">Keluar</option>
            </select>
          </div>

          <div
            v-else
            class="rounded-lg bg-slate-100 p-3 text-sm text-slate-700 dark:bg-slate-800 dark:text-slate-300"
          >
            Tipe transaksi:
            <strong>
              {{ form.kategori === 'Pinjam' ? 'Keluar' : 'Masuk' }}
            </strong>
            <p class="mt-1 text-xs">
              Tipe ditentukan otomatis berdasarkan kategori transaksi.
            </p>
          </div>

          <!-- JUMLAH -->
          <div>
            <label class="mb-1.5 block text-sm font-semibold">
              Jumlah (Rp)
            </label>
            <input
              v-model="form.jumlah"
              type="number"
              min="1"
              step="1"
              required
              placeholder="Contoh: 500000"
              class="w-full rounded-lg border px-3 py-2.5 outline-none focus:border-blue-500 dark:border-slate-700 dark:bg-slate-950"
            />
          </div>

          <!-- INFO JATUH TEMPO OTOMATIS -->
          <div
            v-if="form.kategori === 'Pinjam' && form.tanggal"
            class="rounded-lg bg-amber-50 p-3 text-sm text-amber-800 dark:bg-amber-950/50 dark:text-amber-200"
          >
            <strong>Jatuh tempo otomatis:</strong>
            {{ formatDate(addThreeMonths(form.tanggal)) }}.
            Pinjaman akan ditandai jatuh tempo jika belum lunas pada
            tanggal tersebut.
          </div>

          <!-- ERROR MODAL -->
          <div
            v-if="errorMessage"
            class="rounded-lg bg-red-50 p-3 text-sm text-red-700 dark:bg-red-950/50 dark:text-red-300"
          >
            {{ errorMessage }}
          </div>

          <!-- TOMBOL -->
          <div class="flex flex-col-reverse gap-3 pt-2 sm:flex-row sm:justify-end">
            <button
              type="button"
              class="rounded-lg border px-5 py-2.5 font-semibold dark:border-slate-700"
              :disabled="isSaving"
              @click="closeModal"
            >
              Batal
            </button>

            <button
              type="submit"
              class="rounded-lg bg-blue-600 px-5 py-2.5 font-semibold text-white hover:bg-blue-700 disabled:cursor-not-allowed disabled:opacity-60"
              :disabled="isSaving"
            >
              {{
                isSaving
                  ? 'Menyimpan...'
                  : isEditing
                    ? 'Simpan Perubahan'
                    : 'Simpan Transaksi'
              }}
            </button>
          </div>
        </form>
      </section>
    </div>
  </div>
</template>
