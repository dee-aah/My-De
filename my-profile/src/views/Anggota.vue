
<script setup>
import { ref, computed, onMounted } from 'vue'
import { supabase } from '../supabase'

const anggotaList = ref([])
const loading = ref(false)
const saving = ref(false)
const search = ref('')
const errorMessage = ref('')
const successMessage = ref('')
const isEditing = ref(false)
const showForm = ref(false)

const emptyForm = () => ({
  id: null,
  nama: '',
  no_hp: '',
  alamat: ''
})

const form = ref(emptyForm())

const filteredAnggota = computed(() => {
  const keyword = search.value.trim().toLowerCase()

  if (!keyword) return anggotaList.value

  return anggotaList.value.filter((anggota) =>
    [
      anggota.id,
      anggota.nama,
      anggota.no_hp,
      anggota.alamat
    ].some((value) =>
      String(value ?? '').toLowerCase().includes(keyword)
    )
  )
})

const loadAnggota = async () => {
  loading.value = true
  errorMessage.value = ''

  const { data, error } = await supabase
    .from('anggota')
    .select('id, nama, no_hp, alamat, created_at')
    .order('nama', { ascending: true })

  loading.value = false

  if (error) {
    errorMessage.value = `Gagal mengambil data anggota: ${error.message}`
    return
  }

  anggotaList.value = data ?? []
}

const openAddForm = () => {
  form.value = emptyForm()
  isEditing.value = false
  showForm.value = true
  errorMessage.value = ''
  successMessage.value = ''
}

const editAnggota = (anggota) => {
  form.value = {
    id: anggota.id,
    nama: anggota.nama ?? '',
    no_hp: anggota.no_hp ?? '',
    alamat: anggota.alamat ?? ''
  }

  isEditing.value = true
  showForm.value = true
  errorMessage.value = ''
  successMessage.value = ''
}

const cancelForm = () => {
  showForm.value = false
  form.value = emptyForm()
  isEditing.value = false
  errorMessage.value = ''
}

const saveAnggota = async () => {
  errorMessage.value = ''
  successMessage.value = ''

  const nama = form.value.nama.trim()
  const no_hp = form.value.no_hp.trim()
  const alamat = form.value.alamat.trim()

  if (!nama || !no_hp || !alamat) {
    errorMessage.value = 'Nama, nomor HP, dan alamat wajib diisi.'
    return
  }

  saving.value = true

  // ID tidak dikirim ketika menambah anggota.
  // Supabase akan membuat UUID secara otomatis.
  const payload = { nama, no_hp, alamat }

  let result

  if (isEditing.value) {
    result = await supabase
      .from('anggota')
      .update(payload)
      .eq('id', form.value.id)
      .select()
  } else {
    result = await supabase
      .from('anggota')
      .insert(payload)
      .select()
  }

  saving.value = false

  if (result.error) {
    if (result.error.code === '23505') {
      errorMessage.value = 'Nomor HP tersebut sudah digunakan anggota lain.'
    } else {
      errorMessage.value = `Gagal menyimpan data: ${result.error.message}`
    }
    return
  }

  if (!result.data?.length) {
    errorMessage.value =
      'Data tidak berhasil disimpan atau akses database tidak diizinkan.'
    return
  }

  successMessage.value = isEditing.value
    ? 'Data anggota berhasil diperbarui.'
    : 'Anggota baru berhasil ditambahkan.'

  showForm.value = false
  form.value = emptyForm()
  isEditing.value = false

  await loadAnggota()
}

const deleteAnggota = async (anggota) => {
  const yakin = window.confirm(
    `Hapus anggota "${anggota.nama}"?\nTindakan ini tidak dapat dibatalkan.`
  )

  if (!yakin) return

  errorMessage.value = ''
  successMessage.value = ''

  // Cegah penghapusan jika anggota sudah terhubung
  // dengan transaksi keuangan.
  const { data: transaksi, error: checkError } = await supabase
    .from('keuangan')
    .select('id')
    .eq('anggota_id', anggota.id)
    .limit(1)

  if (checkError && checkError.code !== '42703') {
    errorMessage.value =
      `Gagal memeriksa transaksi anggota: ${checkError.message}`
    return
  }

  if (transaksi?.length) {
    errorMessage.value =
      'Anggota tidak dapat dihapus karena masih terhubung dengan transaksi keuangan.'
    return
  }

  const { error } = await supabase
    .from('anggota')
    .delete()
    .eq('id', anggota.id)

  if (error) {
    errorMessage.value = `Gagal menghapus anggota: ${error.message}`
    return
  }

  successMessage.value = 'Data anggota berhasil dihapus.'
  await loadAnggota()
}

const copyId = async (id) => {
  try {
    await navigator.clipboard.writeText(id)
    successMessage.value = 'ID anggota berhasil disalin.'
  } catch {
    errorMessage.value = `ID anggota: ${id}`
  }
}

onMounted(loadAnggota)
</script>

<template>
  <main class="anggota-page">
    <header class="page-header">
      <div>
        <h1>Data Anggota</h1>
        <p>Kelola data anggota kas BMKG.</p>
      </div>

      <button class="btn btn-primary" @click="openAddForm">
        + Tambah Anggota
      </button>
    </header>

    <div v-if="errorMessage" class="alert alert-error">
      {{ errorMessage }}
      <button class="alert-close" @click="errorMessage = ''">×</button>
    </div>

    <div v-if="successMessage" class="alert alert-success">
      {{ successMessage }}
      <button class="alert-close" @click="successMessage = ''">×</button>
    </div>

    <section v-if="showForm" class="form-card">
      <div class="form-header">
        <h2>{{ isEditing ? 'Edit Anggota' : 'Tambah Anggota' }}</h2>
        <button class="btn btn-light" @click="cancelForm">
          Tutup
        </button>
      </div>

      <form @submit.prevent="saveAnggota">
        <div class="form-grid">
          <div v-if="isEditing" class="form-group full-width">
            <label>ID Anggota</label>
            <div class="id-field">
              <input :value="form.id" readonly />
              <button
                type="button"
                class="btn btn-light"
                @click="copyId(form.id)"
              >
                Salin ID
              </button>
            </div>
            <small>ID dibuat otomatis dan tidak dapat diubah.</small>
          </div>

          <div class="form-group">
            <label for="nama">Nama lengkap</label>
            <input
              id="nama"
              v-model="form.nama"
              type="text"
              placeholder="Masukkan nama anggota"
              maxlength="150"
              required
            />
          </div>

          <div class="form-group">
            <label for="no_hp">Nomor HP</label>
            <input
              id="no_hp"
              v-model="form.no_hp"
              type="tel"
              placeholder="Contoh: 081234567890"
              maxlength="25"
              required
            />
          </div>

          <div class="form-group full-width">
            <label for="alamat">Alamat</label>
            <textarea
              id="alamat"
              v-model="form.alamat"
              placeholder="Masukkan alamat anggota"
              rows="3"
              maxlength="1000"
              required
            />
          </div>
        </div>

        <div class="form-actions">
          <button
            type="button"
            class="btn btn-light"
            @click="cancelForm"
          >
            Batal
          </button>

          <button
            type="submit"
            class="btn btn-primary"
            :disabled="saving"
          >
            {{ saving ? 'Menyimpan...' : isEditing ? 'Simpan Perubahan' : 'Simpan Anggota' }}
          </button>
        </div>
      </form>
    </section>

    <section class="table-card">
      <div class="table-header">
        <div>
          <h2>Daftar Anggota</h2>
          <p>{{ filteredAnggota.length }} anggota ditemukan</p>
        </div>

        <input
          v-model="search"
          class="search-input"
          type="search"
          placeholder="Cari nama, ID, HP, atau alamat..."
        />
      </div>

      <div v-if="loading" class="empty-state">
        Memuat data anggota...
      </div>

      <div v-else-if="filteredAnggota.length === 0" class="empty-state">
        <strong>Belum ada data anggota</strong>
        <p>Tambahkan anggota baru atau ubah kata pencarian.</p>
      </div>

      <div v-else class="table-wrapper">
        <table>
          <thead>
            <tr>
              <th>No.</th>
              <th>ID Anggota</th>
              <th>Nama</th>
              <th>No. HP</th>
              <th>Alamat</th>
              <th>Aksi</th>
            </tr>
          </thead>

          <tbody>
            <tr v-for="(anggota, index) in filteredAnggota" :key="anggota.id">
              <td>{{ index + 1 }}</td>

              <td>
                <button
                  class="id-button"
                  title="Klik untuk menyalin ID"
                  @click="copyId(anggota.id)"
                >
                  {{ anggota.id }}
                </button>
              </td>

              <td class="nama-cell">{{ anggota.nama }}</td>
              <td>{{ anggota.no_hp }}</td>
              <td class="alamat-cell">{{ anggota.alamat }}</td>

              <td>
                <div class="action-buttons">
                  <button
                    class="btn btn-edit"
                    @click="editAnggota(anggota)"
                  >
                    Edit
                  </button>

                  <button
                    class="btn btn-delete"
                    @click="deleteAnggota(anggota)"
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
  </main>
</template>

<style scoped>
.anggota-page {
  width: 100%;
  max-width: 1500px;
  margin: 0 auto;
  padding: 24px;
  color: var(--text-primary, #1f2937);
}

.page-header,
.form-header,
.table-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
}

.page-header {
  margin-bottom: 24px;
}

h1, h2, p {
  margin-top: 0;
}

h1 {
  margin-bottom: 6px;
  font-size: 28px;
  font-weight: 700;
}

h2 {
  margin-bottom: 6px;
  font-size: 18px;
  font-weight: 700;
}

.page-header p,
.table-header p {
  margin-bottom: 0;
  color: var(--text-secondary, #6b7280);
  font-size: 14px;
}

.form-card,
.table-card {
  margin-bottom: 24px;
  padding: 22px;
  border: 1px solid var(--border-color, #e5e7eb);
  border-radius: 14px;
  background: var(--card-bg, #fff);
  box-shadow: 0 3px 12px rgb(0 0 0 / 4%);
}

.form-header {
  margin-bottom: 22px;
}

.form-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 18px;
}

.form-group {
  display: flex;
  min-width: 0;
  flex-direction: column;
  gap: 8px;
}

.full-width {
  grid-column: 1 / -1;
}

.form-group label {
  font-size: 14px;
  font-weight: 600;
}

.form-group input,
.form-group textarea,
.search-input,
.id-field input {
  box-sizing: border-box;
  width: 100%;
  padding: 11px 12px;
  border: 1px solid var(--border-color, #d1d5db);
  border-radius: 8px;
  background: var(--input-bg, #fff);
  color: var(--text-primary, #1f2937);
  font: inherit;
}

.form-group input:focus,
.form-group textarea:focus,
.search-input:focus {
  outline: 2px solid #93c5fd;
  border-color: #2563eb;
}

.form-group small {
  color: var(--text-secondary, #6b7280);
  font-size: 12px;
}

.id-field {
  display: flex;
  gap: 8px;
}

.id-field input {
  flex: 1;
  min-width: 0;
}

.form-actions,
.action-buttons {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}

.form-actions {
  justify-content: flex-end;
  margin-top: 22px;
}

.btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  padding: 9px 13px;
  border: 1px solid transparent;
  border-radius: 8px;
  cursor: pointer;
  font-size: 13px;
  font-weight: 600;
  white-space: nowrap;
}

.btn:disabled {
  cursor: not-allowed;
  opacity: 0.6;
}

.btn-primary {
  background: #2563eb;
  color: white;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-light {
  border-color: var(--border-color, #d1d5db);
  background: var(--card-bg, #fff);
  color: var(--text-primary, #374151);
}

.btn-edit {
  background: #fef3c7;
  color: #92400e;
}

.btn-delete {
  background: #fee2e2;
  color: #b91c1c;
}

.table-header {
  margin-bottom: 20px;
}

.search-input {
  max-width: 360px;
}

.table-wrapper {
  width: 100%;
  overflow-x: auto;
}

table {
  width: 100%;
  border-collapse: collapse;
  font-size: 13px;
}

th,
td {
  padding: 13px 12px;
  border-bottom: 1px solid var(--border-color, #e5e7eb);
  text-align: left;
  vertical-align: middle;
}

th {
  background: var(--table-header-bg, #f9fafb);
  color: var(--text-primary, #374151);
  font-weight: 700;
  white-space: nowrap;
}

tbody tr:hover {
  background: var(--table-hover-bg, #f9fafb);
}

.nama-cell {
  min-width: 150px;
  font-weight: 600;
}

.alamat-cell {
  min-width: 180px;
  max-width: 280px;
  white-space: normal;
  overflow-wrap: anywhere;
}

.id-button {
  max-width: 180px;
  padding: 0;
  border: none;
  background: transparent;
  color: #2563eb;
  cursor: pointer;
  font-family: monospace;
  font-size: 12px;
  overflow-wrap: anywhere;
  text-align: left;
}

.empty-state {
  padding: 42px 16px;
  color: var(--text-secondary, #6b7280);
  text-align: center;
}

.empty-state strong {
  color: var(--text-primary, #374151);
}

.empty-state p {
  margin: 8px 0 0;
  font-size: 14px;
}

.alert {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 12px;
  margin-bottom: 18px;
  padding: 13px 15px;
  border-radius: 9px;
  font-size: 14px;
}

.alert-error {
  background: #fee2e2;
  color: #991b1b;
}

.alert-success {
  background: #dcfce7;
  color: #166534;
}

.alert-close {
  padding: 0 4px;
  border: 0;
  background: transparent;
  color: inherit;
  cursor: pointer;
  font-size: 20px;
}

@media (max-width: 700px) {
  .anggota-page {
    padding: 14px;
  }

  .page-header,
  .table-header {
    align-items: stretch;
    flex-direction: column;
  }

  .page-header .btn {
    align-self: flex-start;
  }

  .form-card,
  .table-card {
    padding: 15px;
  }

  .form-grid {
    grid-template-columns: 1fr;
  }

  .full-width {
    grid-column: auto;
  }

  .search-input {
    max-width: none;
  }
}
</style>
