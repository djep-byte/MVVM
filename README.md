# MVVM
## Membuat Student Registration APP dengan MVVM
### Langkah-langkah
Buat DB ```student_db``` bisa dengan XAMPP dengan menjalankan MySQL kemudian gunakan code berikut
```
CREATE DATABASE IF NOT EXISTS student_db;
USE student_db;

CREATE TABLE IF NOT EXISTS mahasiswa (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nim VARCHAR(20) NOT NULL UNIQUE,
    nama VARCHAR(100) NOT NULL,
    prodi VARCHAR(50),
    jenis_kelamin VARCHAR(20),
    tanggal_lahir DATE,
    alamat VARCHAR(255),
    no_telepon VARCHAR(20)
);
```
Lalu buka terminal dan jalankan code berikut secara berurut
```
dotnet new wpf -n StudentRegistrationMVVM
cd StudentRegistrationMVVM
```
kemudian kita memerlukan ```MySQL Connector``` oleh karena itu install package-nya dengan menjalankan code berikut pada terminal
```
dotnet add package MySqlConnector --version 2.6.2
```
lalu Buat folder Models, lalu di dalamnya buat file ```Mahasiswa.cs``` dan isi dengan code berikut
```
using System;

namespace StudentRegistrationMVVM.Models
{
    public class Mahasiswa
    {
        public int Id { get; set; }
        public string Nim { get; set; } = string.Empty;
        public string Nama { get; set; } = string.Empty;
        public string Prodi { get; set; } = string.Empty;
        public string JenisKelamin { get; set; } = string.Empty;
        public DateTime TanggalLahir { get; set; } = DateTime.Now;
        public string Alamat { get; set; } = string.Empty;
        public string NoTelepon { get; set; } = string.Empty;

        public Mahasiswa Clone()
        {
            return (Mahasiswa)this.MemberwiseClone();
        }
    }
}
```
Kemudian buat folder Data, lalu di dalamnya buat file ```MahasiswaRepository.cs``` dan isi dengan code berikut
```
using System;
using System.Collections.Generic;
using MySqlConnector;
using StudentRegistrationMVVM.Models;

namespace StudentRegistrationMVVM.Data
{
    public class MahasiswaRepository
    {
        private readonly string _connectionString;

        public MahasiswaRepository()
        {
            _connectionString = Environment.GetEnvironmentVariable("STUDENT_DB_CONNECTION_STRING")
                ?? "Server=localhost;Port=3306;Database=student_db;User ID=root;Password=;";
        }

        public List<Mahasiswa> GetAll(string search = "")
        {
            var list = new List<Mahasiswa>();
            using var conn = new MySqlConnection(_connectionString);
            conn.Open();

            string query = "SELECT * FROM mahasiswa WHERE nim LIKE @search OR nama LIKE @search OR prodi LIKE @search ORDER BY id DESC";
            using var cmd = new MySqlCommand(query, conn);
            cmd.Parameters.AddWithValue("@search", $"%{search}%");

            using var reader = cmd.ExecuteReader();
            while (reader.Read())
            {
                list.Add(new Mahasiswa
                {
                    Id = reader.GetInt32("id"),
                    Nim = reader.GetString("nim"),
                    Nama = reader.GetString("nama"),
                    Prodi = reader.IsDBNull(reader.GetOrdinal("prodi")) ? "" : reader.GetString("prodi"),
                    JenisKelamin = reader.IsDBNull(reader.GetOrdinal("jenis_kelamin")) ? "" : reader.GetString("jenis_kelamin"),
                    TanggalLahir = reader.IsDBNull(reader.GetOrdinal("tanggal_lahir")) ? DateTime.Now : reader.GetDateTime("tanggal_lahir"),
                    Alamat = reader.IsDBNull(reader.GetOrdinal("alamat")) ? "" : reader.GetString("alamat"),
                    NoTelepon = reader.IsDBNull(reader.GetOrdinal("no_telepon")) ? "" : reader.GetString("no_telepon")
                });
            }
            return list;
        }

        public bool NimExists(string nim, int excludeId = 0)
        {
            using var conn = new MySqlConnection(_connectionString);
            conn.Open();
            string query = "SELECT COUNT(*) FROM mahasiswa WHERE nim = @nim AND id != @id";
            using var cmd = new MySqlCommand(query, conn);
            cmd.Parameters.AddWithValue("@nim", nim);
            cmd.Parameters.AddWithValue("@id", excludeId);
            return Convert.ToInt32(cmd.ExecuteScalar()) > 0;
        }

        public void Insert(Mahasiswa m)
        {
            using var conn = new MySqlConnection(_connectionString);
            conn.Open();
            string query = @"INSERT INTO mahasiswa (nim, nama, prodi, jenis_kelamin, tanggal_lahir, alamat, no_telepon) 
                            VALUES (@nim, @nama, @prodi, @jenis_kelamin, @tanggal_lahir, @alamat, @no_telepon)";
            using var cmd = new MySqlCommand(query, conn);
            AddParameters(cmd, m);
            cmd.ExecuteNonQuery();
        }

        public void Update(Mahasiswa m)
        {
            using var conn = new MySqlConnection(_connectionString);
            conn.Open();
            string query = @"UPDATE mahasiswa SET nim=@nim, nama=@nama, prodi=@prodi, jenis_kelamin=@jenis_kelamin, 
                            tanggal_lahir=@tanggal_lahir, alamat=@alamat, no_telepon=@no_telepon WHERE id=@id";
            using var cmd = new MySqlCommand(query, conn);
            AddParameters(cmd, m);
            cmd.Parameters.AddWithValue("@id", m.Id);
            cmd.ExecuteNonQuery();
        }

        public void Delete(int id)
        {
            using var conn = new MySqlConnection(_connectionString);
            conn.Open();
            string query = "DELETE FROM mahasiswa WHERE id = @id";
            using var cmd = new MySqlCommand(query, conn);
            cmd.Parameters.AddWithValue("@id", id);
            cmd.ExecuteNonQuery();
        }

        private static void AddParameters(MySqlCommand cmd, Mahasiswa m)
        {
            cmd.Parameters.AddWithValue("@nim", m.Nim);
            cmd.Parameters.AddWithValue("@nama", m.Nama);
            cmd.Parameters.AddWithValue("@prodi", m.Prodi);
            cmd.Parameters.AddWithValue("@jenis_kelamin", m.JenisKelamin);
            cmd.Parameters.AddWithValue("@tanggal_lahir", m.TanggalLahir);
            cmd.Parameters.AddWithValue("@alamat", m.Alamat);
            cmd.Parameters.AddWithValue("@no_telepon", m.NoTelepon);
        }
    }
}
```
Buat folder ViewModels, lalu di dalamnya buat file ```RelayCommand.cs``` dan ```MahasiswaViewModel.cs``` isi dengan code ini  

code ```RelayCommand.cs```
```
using System;
using System.Windows.Input;

namespace StudentRegistrationMVVM.ViewModels
{
    public class RelayCommand : ICommand
    {
        private readonly Action<object?> _execute;
        private readonly Predicate<object?>? _canExecute;

        public RelayCommand(Action<object?> execute, Predicate<object?>? canExecute = null)
        {
            _execute = execute ?? throw new ArgumentNullException(nameof(execute));
            _canExecute = canExecute;
        }

        public bool CanExecute(object? parameter) => _canExecute == null || _canExecute(parameter);

        public void Execute(object? parameter) => _execute(parameter);

        public event EventHandler? CanExecuteChanged
        {
            add => CommandManager.RequerySuggested += value;
            remove => CommandManager.RequerySuggested -= value;
        }
    }
}
```
code ```MahasiswaViewModel.cs```
```
using System;
using System.Collections.ObjectModel;
using System.ComponentModel;
using System.Linq;
using System.Runtime.CompilerServices;
using System.Windows.Input;
using StudentRegistrationMVVM.Data;
using StudentRegistrationMVVM.Models;

namespace StudentRegistrationMVVM.ViewModels
{
    public class MahasiswaViewModel : INotifyPropertyChanged
    {
        private readonly MahasiswaRepository _repository;
        private Mahasiswa _formMahasiswa;
        private Mahasiswa? _selectedMahasiswa;
        private string _searchKeyword = string.Empty;
        private string _statusMessage = string.Empty;

        public ObservableCollection<Mahasiswa> ListMahasiswa { get; set; }

        public Mahasiswa FormMahasiswa
        {
            get => _formMahasiswa;
            set { _formMahasiswa = value; OnPropertyChanged(); }
        }

        public Mahasiswa? SelectedMahasiswa
        {
            get => _selectedMahasiswa;
            set
            {
                _selectedMahasiswa = value;
                OnPropertyChanged();
                if (_selectedMahasiswa != null)
                {
                    FormMahasiswa = _selectedMahasiswa.Clone();
                }
            }
        }

        public string SearchKeyword
        {
            get => _searchKeyword;
            set { _searchKeyword = value; OnPropertyChanged(); }
        }

        public string StatusMessage
        {
            get => _statusMessage;
            set { _statusMessage = value; OnPropertyChanged(); }
        }

        public int TotalMahasiswa => ListMahasiswa.Count;

        public ICommand SaveCommand { get; }
        public ICommand DeleteCommand { get; }
        public ICommand ResetCommand { get; }
        public ICommand SearchCommand { get; }

        public MahasiswaViewModel()
        {
            _repository = new MahasiswaRepository();
            ListMahasiswa = new ObservableCollection<Mahasiswa>();
            _formMahasiswa = new Mahasiswa();

            SaveCommand = new RelayCommand(_ => Save());
            DeleteCommand = new RelayCommand(_ => Delete(), _ => SelectedMahasiswa != null);
            ResetCommand = new RelayCommand(_ => ResetForm());
            SearchCommand = new RelayCommand(_ => LoadData());

            LoadData();
        }

        public void LoadData()
        {
            try
            {
                var data = _repository.GetAll(SearchKeyword);
                ListMahasiswa.Clear();
                foreach (var item in data) ListMahasiswa.Add(item);
                OnPropertyChanged(nameof(TotalMahasiswa));
                StatusMessage = "Data berhasil dimuat.";
            }
            catch (Exception ex)
            {
                StatusMessage = $"Koneksi/Error: {ex.Message}";
            }
        }

        private bool ValidateInput()
        {
            if (string.IsNullOrWhiteSpace(FormMahasiswa.Nim) || FormMahasiswa.Nim.Length < 8 || FormMahasiswa.Nim.Length > 12 || !FormMahasiswa.Nim.All(char.IsDigit))
            {
                StatusMessage = "Validasi Gagal: NIM harus 8-12 digit angka!";
                return false;
            }
            if (string.IsNullOrWhiteSpace(FormMahasiswa.Nama) || FormMahasiswa.Nama.Length < 3)
            {
                StatusMessage = "Validasi Gagal: Nama minimal 3 karakter!";
                return false;
            }
            int age = DateTime.Now.Year - FormMahasiswa.TanggalLahir.Year;
            if (FormMahasiswa.TanggalLahir > DateTime.Now.AddYears(-age)) age--;
            if (age < 15)
            {
                StatusMessage = "Validasi Gagal: Usia mahasiswa minimal 15 tahun!";
                return false;
            }
            if (string.IsNullOrWhiteSpace(FormMahasiswa.Alamat) || FormMahasiswa.Alamat.Length < 5)
            {
                StatusMessage = "Validasi Gagal: Alamat minimal 5 karakter!";
                return false;
            }
            if (string.IsNullOrWhiteSpace(FormMahasiswa.NoTelepon) || (!FormMahasiswa.NoTelepon.StartsWith("08") && !FormMahasiswa.NoTelepon.StartsWith("628")))
            {
                StatusMessage = "Validasi Gagal: No Telepon harus diawali '08' atau '628'!";
                return false;
            }
            if (_repository.NimExists(FormMahasiswa.Nim, FormMahasiswa.Id))
            {
                StatusMessage = "Validasi Gagal: NIM sudah digunakan oleh mahasiswa lain!";
                return false;
            }
            return true;
        }

        private void Save()
        {
            if (!ValidateInput()) return;

            try
            {
                if (FormMahasiswa.Id == 0)
                {
                    _repository.Insert(FormMahasiswa);
                    StatusMessage = "Data mahasiswa berhasil ditambahkan!";
                }
                else
                {
                    _repository.Update(FormMahasiswa);
                    StatusMessage = "Data mahasiswa berhasil diperbarui!";
                }
                ResetForm();
                LoadData();
            }
            catch (Exception ex)
            {
                StatusMessage = $"Gagal menyimpan: {ex.Message}";
            }
        }

        private void Delete()
        {
            if (SelectedMahasiswa == null) return;
            try
            {
                _repository.Delete(SelectedMahasiswa.Id);
                StatusMessage = "Data berhasil dihapus.";
                ResetForm();
                LoadData();
            }
            catch (Exception ex)
            {
                StatusMessage = $"Gagal menghapus: {ex.Message}";
            }
        }

        private void ResetForm()
        {
            FormMahasiswa = new Mahasiswa();
            SelectedMahasiswa = null;
        }

        public event PropertyChangedEventHandler? PropertyChanged;
        protected void OnPropertyChanged([CallerMemberName] string? name = null)
        {
            PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(name));
        }
    }
}
```
kemudian buka file ```MainWindow.xaml``` dan isi dengan code berikut
```
<Window x:Class="StudentRegistrationMVVM.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="Student Registration" Height="650" Width="1000" WindowStartupLocation="CenterScreen">
    <Grid Margin="15">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
            <RowDefinition Height="Auto"/>
        </Grid.RowDefinitions>

        <!-- Header Bar -->
        <Border Grid.Row="0" Background="#1E40AF" Padding="15" CornerRadius="5" Margin="0,0,0,15">
            <StackPanel>
                <TextBlock Text="STUDENT REGISTRATION" Foreground="White" FontSize="20" FontWeight="Bold"/>
                <TextBlock Text="Pengelolaan data mahasiswa dengan pola MVVM" Foreground="#E0E7FF" FontSize="12"/>
            </StackPanel>
        </Border>

        <!-- Content Area -->
        <Grid Grid.Row="1">
            <Grid.ColumnDefinitions>
                <ColumnDefinition Width="320"/>
                <ColumnDefinition Width="*"/>
            </Grid.ColumnDefinitions>

            <!-- Left Panel: Form Input -->
            <Border Grid.Column="0" Background="#F8FAFC" BorderBrush="#E2E8F0" BorderThickness="1" CornerRadius="5" Padding="15" Margin="0,0,10,0">
                <ScrollViewer VerticalScrollBarVisibility="Auto">
                    <StackPanel>
                        <TextBlock Text="FORM MAHASISWA" FontWeight="Bold" FontSize="14" Margin="0,0,0,15"/>

                        <TextBlock Text="NIM"/>
                        <TextBox Text="{Binding FormMahasiswa.Nim, UpdateSourceTrigger=PropertyChanged}" Margin="0,2,0,10" Padding="5"/>

                        <TextBlock Text="Nama mahasiswa"/>
                        <TextBox Text="{Binding FormMahasiswa.Nama, UpdateSourceTrigger=PropertyChanged}" Margin="0,2,0,10" Padding="5"/>

                        <TextBlock Text="Program studi"/>
                        <ComboBox Text="{Binding FormMahasiswa.Prodi, UpdateSourceTrigger=PropertyChanged}" IsEditable="True" Margin="0,2,0,10" Padding="5">
                            <ComboBoxItem Content="Teknik Informatika"/>
                            <ComboBoxItem Content="Sistem Informasi"/>
                            <ComboBoxItem Content="Teknik Komputer"/>
                            <ComboBoxItem Content="Data Science"/>
                        </ComboBox>

                        <TextBlock Text="Jenis kelamin"/>
                        <ComboBox Text="{Binding FormMahasiswa.JenisKelamin, UpdateSourceTrigger=PropertyChanged}" IsEditable="True" Margin="0,2,0,10" Padding="5">
                            <ComboBoxItem Content="Laki-laki"/>
                            <ComboBoxItem Content="Perempuan"/>
                        </ComboBox>

                        <TextBlock Text="Tanggal lahir"/>
                        <DatePicker SelectedDate="{Binding FormMahasiswa.TanggalLahir, UpdateSourceTrigger=PropertyChanged}" Margin="0,2,0,10" Padding="5"/>

                        <TextBlock Text="Alamat"/>
                        <TextBox Text="{Binding FormMahasiswa.Alamat, UpdateSourceTrigger=PropertyChanged}" TextWrapping="Wrap" AcceptsReturn="True" Height="60" Margin="0,2,0,10" Padding="5"/>

                        <TextBlock Text="Nomor telepon"/>
                        <TextBox Text="{Binding FormMahasiswa.NoTelepon, UpdateSourceTrigger=PropertyChanged}" Margin="0,2,0,15" Padding="5"/>

                        <Grid Margin="0,10,0,0">
                            <Grid.ColumnDefinitions>
                                <ColumnDefinition Width="*"/>
                                <ColumnDefinition Width="Auto"/>
                            </Grid.ColumnDefinitions>
                            <Button Content="Simpan / Update" Command="{Binding SaveCommand}" Background="#2563EB" Foreground="White" Padding="10,6" Margin="0,0,5,0" FontWeight="SemiBold"/>
                            <Button Grid.Column="1" Content="Reset" Command="{Binding ResetCommand}" Padding="10,6"/>
                        </Grid>
                    </StackPanel>
                </ScrollViewer>
            </Border>

            <!-- Right Panel: Data Table & Search -->
            <Grid Grid.Column="1" Margin="10,0,0,0">
                <Grid.RowDefinitions>
                    <RowDefinition Height="Auto"/>
                    <RowDefinition Height="*"/>
                    <RowDefinition Height="Auto"/>
                </Grid.RowDefinitions>

                <DockPanel Grid.Row="0" Margin="0,0,0,10">
                    <TextBlock Text="DATA MAHASISWA" FontWeight="Bold" FontSize="14" VerticalAlignment="Center"/>
                    <StackPanel Orientation="Horizontal" DockPanel.Dock="Right" HorizontalAlignment="Right">
                        <TextBox Width="200" Text="{Binding SearchKeyword, UpdateSourceTrigger=PropertyChanged}" Padding="5" Margin="0,0,5,0"/>
                        <Button Content="Cari" Command="{Binding SearchCommand}" Padding="10,5" Margin="0,0,5,0"/>
                        <Button Content="Hapus" Command="{Binding DeleteCommand}" Background="#DC2626" Foreground="White" Padding="10,5"/>
                    </StackPanel>
                </DockPanel>

                <!-- DataGrid Read-Only -->
                <DataGrid Grid.Row="1" ItemsSource="{Binding ListMahasiswa}" SelectedItem="{Binding SelectedMahasiswa}" 
                          AutoGenerateColumns="False" IsReadOnly="True" SelectionMode="Single"
                          GridLinesVisibility="Horizontal" HeadersVisibility="Column" Background="White">
                    <DataGrid.Columns>
                        <DataGridTextColumn Header="NIM" Binding="{Binding Nim}" Width="100"/>
                        <DataGridTextColumn Header="Nama" Binding="{Binding Nama}" Width="150"/>
                        <DataGridTextColumn Header="Prodi" Binding="{Binding Prodi}" Width="130"/>
                        <DataGridTextColumn Header="Gender" Binding="{Binding JenisKelamin}" Width="90"/>
                        <DataGridTextColumn Header="Tanggal Lahir" Binding="{Binding TanggalLahir, StringFormat=\{0:dd/MM/yyyy\}}" Width="100"/>
                        <DataGridTextColumn Header="No Telp" Binding="{Binding NoTelepon}" Width="110"/>
                        <DataGridTextColumn Header="Alamat" Binding="{Binding Alamat}" Width="*"/>
                    </DataGrid.Columns>
                </DataGrid>

                <!-- Footer Summary -->
                <StackPanel Grid.Row="2" Orientation="Horizontal" Margin="0,10,0,0">
                    <TextBlock Text="Jumlah mahasiswa: " FontWeight="SemiBold"/>
                    <TextBlock Text="{Binding TotalMahasiswa}" FontWeight="Bold"/>
                </StackPanel>
            </Grid>
        </Grid>

        <!-- Status Message Bar -->
        <Border Grid.Row="2" Background="#F1F5F9" Padding="8,5" Margin="0,10,0,0" CornerRadius="3">
            <TextBlock Text="{Binding StatusMessage}" Foreground="#334155" FontSize="11" FontStyle="Italic"/>
        </Border>
    </Grid>
</Window>
```
buka file ```MainWindow.xaml.cs``` dan isi dengan code berikut
```
using System.Windows;
using StudentRegistrationMVVM.ViewModels;

namespace StudentRegistrationMVVM
{
    public partial class MainWindow : Window
    {
        public MainWindow()
        {
            InitializeComponent();
            DataContext = new MahasiswaViewModel();
        }
    }
}
```
untuk menjalankan program pastikan MySQL di XAMPP masih pada posisi start, kemudian buka terminal pada vs code dan jalankan code berikut  
```
dotnet run
```
## Dokumentasi

### Hasil Run
<img width="734" height="476" alt="image" src="https://github.com/user-attachments/assets/78619933-110b-437a-9c34-929aa445a3bb" />

### Menambahkan Data Mahasiswa
<img width="1277" height="503" alt="image" src="https://github.com/user-attachments/assets/499da6a8-cbb7-47b4-a70d-3679f592fc82" />

### Mencari Data
<img width="973" height="65" alt="image" src="https://github.com/user-attachments/assets/9dcae38c-b561-4ced-bf3e-6d2f3739d90f" />

### Mengubah dan Menghapus Data
<img width="1253" height="422" alt="image" src="https://github.com/user-attachments/assets/6faf298d-870b-4eef-aedc-99b5f9ca651c" />
