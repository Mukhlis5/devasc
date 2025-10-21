# devasc
Program python ketika dijalankan langsung mendisable network yang dijalankan

#!/usr/bin/env python3
"""
Network Management Tool for DEVASC VM
Program untuk menonaktifkan/mengaktifkan interface jaringan di DEVASC VM
"""

import subprocess
import sys
import time

def run_command(cmd, require_sudo=False):
    """Menjalankan command dan mengembalikan output"""
    try:
        if require_sudo:
            cmd = ['sudo'] + cmd
        result = subprocess.run(cmd, capture_output=True, text=True, check=True)
        return result.stdout.strip()
    except subprocess.CalledProcessError as e:
        print(f"Error menjalankan command: {e}")
        return None

def get_network_interfaces():
    """Mendapatkan daftar interface jaringan"""
    print("\n📡 Mendapatkan informasi interface jaringan...")
    interfaces = []
    
    # Gunakan ip addr seperti di lab
    output = run_command(['ip', 'addr'])
    if output:
        for line in output.split('\n'):
            if ':' in line and not 'link/loopback' in line:
                parts = line.split(':')
                if len(parts) >= 2:
                    interface = parts[1].strip()
                    if interface and interface != 'lo':  # Skip loopback
                        interfaces.append(interface)
    
    return interfaces

def display_interface_status():
    """Menampilkan status semua interface"""
    print("\n" + "="*50)
    print("🖧 STATUS INTERFACE JARINGAN DEVASC VM")
    print("="*50)
    
    interfaces = get_network_interfaces()
    if not interfaces:
        print("Tidak ada interface jaringan yang ditemukan")
        return
    
    for interface in interfaces:
        # Cek status interface
        output = run_command(['ip', 'addr', 'show', interface])
        if output:
            status = "UP" if "state UP" in output else "DOWN"
            ip_address = "Tidak ada IP"
            
            # Extract IP address
            for line in output.split('\n'):
                if 'inet ' in line:
                    ip_parts = line.strip().split()
                    if len(ip_parts) >= 2:
                        ip_address = ip_parts[1]
                        break
            
            print(f"🔹 {interface:10} | Status: {status:4} | IP: {ip_address}")

def disable_interface(interface_name):
    """Menonaktifkan interface tertentu"""
    print(f"\n⚠️  Menonaktifkan interface: {interface_name}")
    
    # Konfirmasi untuk interface penting
    if interface_name == 'enp0s3':
        confirm = input("PERINGATAN: Interface enp0s3 adalah interface utama! Lanjutkan? (y/N): ")
        if confirm.lower() != 'y':
            print("Operasi dibatalkan.")
            return False
    
    try:
        # Gunakan ifconfig seperti di lab
        result = run_command(['ifconfig', interface_name, 'down'], require_sudo=True)
        if result is not None:
            print(f"✅ Interface {interface_name} berhasil dinonaktifkan")
            
            # Verifikasi status
            time.sleep(1)
            output = run_command(['ip', 'addr', 'show', interface_name])
            if output and "state DOWN" in output:
                print(f"✅ Verifikasi: {interface_name} status DOWN")
            return True
        else:
            print(f"❌ Gagal menonaktifkan {interface_name}")
            return False
            
    except Exception as e:
        print(f"❌ Error: {e}")
        return False

def enable_interface(interface_name):
    """Mengaktifkan interface tertentu"""
    print(f"\n🔄 Mengaktifkan interface: {interface_name}")
    
    try:
        result = run_command(['ifconfig', interface_name, 'up'], require_sudo=True)
        if result is not None:
            print(f"✅ Interface {interface_name} berhasil diaktifkan")
            
            # Verifikasi status
            time.sleep(2)
            output = run_command(['ip', 'addr', 'show', interface_name])
            if output and "state UP" in output:
                print(f"✅ Verifikasi: {interface_name} status UP")
                
                # Tampilkan IP address baru
                for line in output.split('\n'):
                    if 'inet ' in line:
                        ip_parts = line.strip().split()
                        if len(ip_parts) >= 2:
                            print(f"📡 IP Address: {ip_parts[1]}")
            return True
        else:
            print(f"❌ Gagal mengaktifkan {interface_name}")
            return False
            
    except Exception as e:
        print(f"❌ Error: {e}")
        return False

def test_connectivity():
    """Test konektivitas jaringan"""
    print("\n🌐 Testing konektivitas...")
    
    # Test ping ke gateway
    print("Pinging gateway (8.8.8.8)...")
    try:
        result = subprocess.run(['ping', '-c', '3', '8.8.8.8'], 
                              capture_output=True, text=True, timeout=10)
        if result.returncode == 0:
            print("✅ Koneksi internet: OK")
        else:
            print("❌ Koneksi internet: GAGAL")
    except:
        print("❌ Koneksi internet: GAGAL")
    
    # Test DNS
    print("Testing DNS resolution...")
    try:
        result = subprocess.run(['nslookup', 'www.cisco.com'], 
                              capture_output=True, text=True, timeout=5)
        if result.returncode == 0:
            print("✅ DNS resolution: OK")
        else:
            print("❌ DNS resolution: GAGAL")
    except:
        print("❌ DNS resolution: GAGAL")

def main():
    """Menu utama"""
    print("🚀 DEVASC VM Network Management Tool")
    print("   Berdasarkan Lab Network Troubleshooting Tools")
    print("   Tools: ifconfig, ip, ping, nslookup")
    
    while True:
        print("\n" + "="*50)
        print("📋 MENU UTAMA")
        print("="*50)
        print("1. Tampilkan status interface")
        print("2. Nonaktifkan interface")
        print("3. Aktifkan interface") 
        print("4. Test konektivitas")
        print("5. Keluar")
        print("-"*50)
        
        choice = input("Pilih opsi (1-5): ").strip()
        
        if choice == '1':
            display_interface_status()
            
        elif choice == '2':
            interfaces = get_network_interfaces()
            if interfaces:
                print("\nPilih interface untuk dinonaktifkan:")
                for i, interface in enumerate(interfaces, 1):
                    print(f"  {i}. {interface}")
                
                try:
                    selection = int(input("Masukkan nomor: ")) - 1
                    if 0 <= selection < len(interfaces):
                        disable_interface(interfaces[selection])
                    else:
                        print("❌ Pilihan tidak valid")
                except ValueError:
                    print("❌ Input harus angka")
            else:
                print("❌ Tidak ada interface yang ditemukan")
                
        elif choice == '3':
            interfaces = get_network_interfaces()
            if interfaces:
                print("\nPilih interface untuk diaktifkan:")
                for i, interface in enumerate(interfaces, 1):
                    print(f"  {i}. {interface}")
                
                try:
                    selection = int(input("Masukkan nomor: ")) - 1
                    if 0 <= selection < len(interfaces):
                        enable_interface(interfaces[selection])
                    else:
                        print("❌ Pilihan tidak valid")
                except ValueError:
                    print("❌ Input harus angka")
            else:
                print("❌ Tidak ada interface yang ditemukan")
                
        elif choice == '4':
            test_connectivity()
            
        elif choice == '5':
            print("\n👋 Keluar dari program...")
            print("Tip: Gunakan command berikut untuk troubleshooting:")
            print("  - ip addr show")
            print("  - ping 8.8.8.8") 
            print("  - nslookup www.cisco.com")
            print("  - traceroute www.netacad.com")
            break
            
        else:
            print("❌ Pilihan tidak valid")

if __name__ == "__main__":
    # Check if running in Linux environment
    try:
        with open('/etc/os-release', 'r') as f:
            if 'DEVASC' in f.read() or 'Ubuntu' in f.read():
                print("✅ Environment DEVASC terdeteksi")
            else:
                print("⚠️  Environment tidak dikenali, lanjutkan dengan hati-hati")
    except:
        print("⚠️  Tidak dapat memverifikasi environment")
    
    main()
