# BSM201 Proje
# Proje Konusu: Muhasebe Uygulaması
# Ekip Üyesi: Ahmet Çoban(220101036)
import json
import tkinter as tk
from tkinter import messagebox
import os

class Node:
    def __init__(self, company_name):
        self.company_name = company_name
        self.next = None
        self.own_tree_root = RootOfOwnCompany(company_name)

class CompanyLinkedList:
    def __init__(self):
        self.first_node = None
        self.existing_companies = set()
        self.load_from_file('companynames.txt')  # Program başladığında mevcut şirketleri yükle

    def add_company(self, company_name):
        if company_name not in self.existing_companies:
            new_company = Node(company_name)
            if self.first_node is None:
                self.first_node = new_company
            else:
                temp = self.first_node
                while temp.next is not None:
                    temp = temp.next
                temp.next = new_company

            # Şirket listesini dosyaya yaz
            self.save_to_file('companynames.txt')

            # Şirketi sete ekle
            self.existing_companies.add(company_name)
        else:
            messagebox.showwarning("Hata", f"{company_name} adlı şirket zaten var.")

    def delete_company(self, company_name):
        temp = self.first_node
        previous_node = None

        while temp:
            if temp.company_name == company_name:
                if previous_node:
                    previous_node.next = temp.next
                else:
                    self.first_node = temp.next

                self.save_to_file('companynames.txt')
                break

            previous_node, temp = temp, temp.next

    def list_companies(self):
        temp = self.first_node
        while temp:
            print(temp.company_name)
            temp = temp.next
        print()

    def save_to_file(self, filename):
        company_list = []  

        temp = self.first_node
        while temp is not None:
            company_list.append(temp.company_name)
            temp = temp.next

        try:
            with open(filename, 'w') as file:
                for company_name in company_list:
                    file.write(company_name + '\n')
        except Exception as e:
            print(f"Hata: {e}")

    def load_from_file(self, filename):
        self.first_node = None

        try:
            with open(filename, 'r') as file:
                company_names = file.read().splitlines()
        except FileNotFoundError:
            company_names = []

        for company_name in company_names:
            self.add_company(company_name)
            self.existing_companies.add(company_name)

    def __iter__(self):
        temp = self.first_node
        while temp:
            yield temp
            temp = temp.next

class RootOfOwnCompany:
    def __init__(self, company_name):
        self.company_name = company_name
        self.contact_informations = ContactInformation()
        self.customer_informations = CustomerInformation()
        self.employee_informations = EmployeeInformation()
        self.financial_informations = FinancialInformation()

    def save_to_file(self, filename):
        with open(filename, 'w') as file:
            json.dump({
                "Company Name": self.company_name,
                "Contact Information": self.contact_informations.__dict__,
                "Customer Information": self.customer_informations.__dict__,
                "Employee Information": self.employee_informations.__dict__,
                "Financial Information": self.financial_informations.to_json(),
            }, file, default=lambda x: x.__dict__)

    def load_from_file(self, filename):
        with open(filename, 'r') as file:
            data = json.load(file)
            self.company_name = data["Company Name"]
            for info in (self.contact_informations, self.customer_informations, self.employee_informations, self.financial_informations):
                info.__dict__.update(data[info.__class__.__name__])

class InformationBase:
    def save_to_file(self, filename):
        with open(filename, 'w') as file:
            json.dump(self.__dict__, file)

    def load_from_file(self, filename):
        with open(filename, 'r') as file:
            data = json.load(file)
            self.__dict__.update(data)

class ContactInformation(InformationBase):
    def __init__(self):
        self.address, self.phone_number, self.mail_address = None, None, None

class CustomerInformation(InformationBase):
    def __init__(self):
        self.customer_name, self.customer_surname, self.customer_phone_number, self.customer_address = None, None, None, None

class EmployeeInformation(InformationBase):
    employee_ID, employees, financial_info = 1, [], None

    def __init__(self):
        self.employee_name, self.employee_surname, self.employee_salary, self.is_terminated = None, None, 0, False
        self.employee_ID = EmployeeInformation.employee_ID
        EmployeeInformation.employee_ID += 1
        EmployeeInformation.employees.append(self)
        self.financial_info = FinancialInformation()

    def set_employee_information(self, name, surname, salary):
        self.employee_name, self.employee_surname, self.employee_salary = name, surname, salary

    def get_employee_full_name(self):
        return f"{self.employee_name} {self.employee_surname}"

    def get_employee_salary(self):
        return self.employee_salary

    def promote_employee(self, salary_increase):
        self.employee_salary += salary_increase

    def terminate_employee(self):
        self.is_terminated = True

    def get_employee_information(self):
        return {"ID": self.employee_ID, "Name": self.get_employee_full_name(), "Salary": self.get_employee_salary(), "IsTerminated": self.is_terminated}

    @classmethod
    def get_employee_by_id(cls, employee_id):
        return next((employee for employee in cls.employees if employee.employee_ID == employee_id), None)

class FinancialInformation(InformationBase):
    def __init__(self):
        self.income, self.payment, self.sales = 0, 0, 0

    def to_json(self):
        return {
            "income": self.income,
            "payment": self.payment,
            "sales": self.sales,
        }

def load_company_information(filename):
    try:
        with open(filename, 'r') as file:
            data = json.load(file)

            company_name = data.get("Company Name", "")
            if not company_name:
                print(f"Hata: {filename} dosyasında şirket adı bulunamadı.")
                return None

            contact_info = ContactInformation()
            contact_info.__dict__.update(data.get("Contact Information", {}))

            customer_info = CustomerInformation()
            customer_info.__dict__.update(data.get("Customer Information", {}))

            employee_info = EmployeeInformation()
            employee_info.__dict__.update(data.get("Employee Information", {}))

            financial_info = FinancialInformation()
            financial_info.__dict__.update(data.get("Financial Information", {}))

            return CompanyInformation(company_name, contact_info, customer_info, employee_info, financial_info)
    except Exception as e:
        print(f"Hata: {e}")
        return None

class CompanyInformation:
    def __init__(self, company_name, contact_info=None, customer_info=None, employee_info=None, financial_info=None):
        self.company_name = company_name
        self.contact_info = contact_info or ContactInformation()
        self.customer_info = customer_info or CustomerInformation()
        self.employee_info = employee_info or EmployeeInformation()
        self.financial_info = financial_info or FinancialInformation()

class CompanyManagement(tk.Tk):
    def __init__(self):
        super().__init__()
        self.title("COMPANY MANAGEMENT SYSTEM")
        self.geometry("500x500")

        self.company_linked_list = CompanyLinkedList()
        self.main_menu()

    def main_menu(self):
        tk.Label(self, text="COMPANY MANAGEMENT SYSTEM", font=("Arciform", 16)).pack(pady=20)
        tk.Button(self, text="Yeni Şirket Oluştur", command=self.new_company_window).pack(pady=10)
        tk.Button(self, text="Şirket Sil", command=self.delete_company_window).pack(pady=10)
        tk.Button(self, text="Şirket Listesi", command=self.show_companies).pack(pady=10)

    def new_company_window(self):
        new_company_window = tk.Toplevel(self)
        new_company_window.title("YENİ ŞİRKET OLUŞTUR")
        tk.Label(new_company_window, text="YENİ ŞİRKET OLUŞTUR", font=("Arciform", 16)).pack(pady=20)
        tk.Label(new_company_window, text="Şirket Adı:").pack()
        company_name_entry = tk.Entry(new_company_window)
        company_name_entry.pack(pady=10)
        tk.Button(new_company_window, text="Oluştur", command=lambda: self.create_new_company(company_name_entry.get())).pack(pady=10)

    def create_new_company(self, company_name):
        if company_name:
            # Yeni şirket oluştur
            self.company_linked_list.add_company(company_name)

            # Kullanıcıdan şirket detaylarına ait bilgileri al
            contact_info = self.get_contact_information_from_user()
            customer_info = self.get_customer_information_from_user()
            employee_info = self.get_employee_information_from_user()
            financial_info = self.get_financial_information_from_user()

            # Şirketin detaylarını içeren RootOfOwnCompany objesini oluştur
            root_of_own_company = RootOfOwnCompany(company_name)
            root_of_own_company.contact_informations = contact_info
            root_of_own_company.customer_informations = customer_info
            root_of_own_company.employee_informations = employee_info
            root_of_own_company.financial_informations = financial_info

            # RootOfOwnCompany objesini dosyaya kaydet
            root_of_own_company.save_to_file(f'{company_name}.txt')

            # Bilgileri ekledikten sonra mesaj kutusu göster
            messagebox.showinfo("Başarılı", f"{company_name} adlı şirket oluşturuldu ve detayları kaydedildi.")
        else:
            messagebox.showwarning("Hata", "Lütfen geçerli bir şirket adı girin.")

    def get_contact_information_from_user(self):
        contact_info = ContactInformation()

        # Kullanıcıdan iletişim bilgilerini al
        contact_info.address = input("Adres: ")
        contact_info.phone_number = input("Telefon Numarası: ")
        contact_info.mail_address = input("E-posta Adresi: ")

        return contact_info

    def get_customer_information_from_user(self):
        customer_info = CustomerInformation()

        # Kullanıcıdan müşteri bilgilerini al
        customer_info.customer_name = input("Müşteri Adı: ")
        customer_info.customer_surname = input("Müşteri Soyadı: ")
        customer_info.customer_phone_number = input("Müşteri Telefon Numarası: ")
        customer_info.customer_address = input("Müşteri Adresi: ")

        return customer_info

    def get_employee_information_from_user(self):
        employee_info = EmployeeInformation()

        # Kullanıcıdan çalışan bilgilerini al
        employee_info.employee_name = input("Çalışan Adı: ")
        employee_info.employee_surname = input("Çalışan Soyadı: ")

        # Çalışan maaşını sayısal bir değer olarak al
        try:
            employee_info.employee_salary = float(input("Çalışan Maaşı: "))
        except ValueError:
            messagebox.showwarning("Hata", "Lütfen geçerli bir sayısal değer girin.")
            return None

        return employee_info

    def get_financial_information_from_user(self):
        financial_info = FinancialInformation()

        # Kullanıcıdan finansal bilgileri al
        try:
            financial_info.income = float(input("Gelir: "))
            financial_info.payment = float(input("Ödeme: "))
            financial_info.sales = float(input("Satış: "))
        except ValueError:
            messagebox.showwarning("Hata", "Lütfen geçerli sayısal değerler girin.")
            return None

        return financial_info

    def delete_company_window(self):
        delete_company_window = tk.Toplevel(self)
        delete_company_window.title("ŞİRKET SİL")
        tk.Label(delete_company_window, text="ŞİRKET SİL", font=("Arciform", 16)).pack(pady=20)
        tk.Label(delete_company_window, text="Şirket Adı:").pack()
        company_name_entry = tk.Entry(delete_company_window)
        company_name_entry.pack(pady=10)
        tk.Button(delete_company_window, text="Sil", command=lambda: self.delete_company(company_name_entry.get())).pack(pady=10)

    def delete_company(self, company_name):
        if company_name in self.company_linked_list.existing_companies:
            file_path = f"{company_name}.txt"
            try:
                # İlgili şirketin özel dosyasını sil
                os.remove(file_path)
                messagebox.showinfo("Başarılı", f"{company_name} adlı şirket silindi.")
            except FileNotFoundError:
                messagebox.showwarning("Hata", f"{file_path} dosyası bulunamadı.")
            
            # Şirketi listeden sil
            self.company_linked_list.delete_company(company_name)
        else:
            messagebox.showwarning("Hata", "Belirtilen isimde bir şirket bulunamadı.")
    
    def show_companies(self):
        show_companies_window = tk.Toplevel(self)
        show_companies_window.title("ŞİRKETLER")

        label = tk.Label(show_companies_window, text="ŞİRKETLER", font=("Arciform", 16))
        label.pack(pady=20)

        text_widget = tk.Text(show_companies_window, wrap=tk.WORD, font=("Arial", 12), height=20, width=80)
        text_widget.pack()

        for company_name in self.company_linked_list.existing_companies:
            # Her şirketin dosya adını kullanarak doğru dosyayı oku
            company_info = load_company_information(f"{company_name}.txt")

            if company_info:
                # Dosyadan yüklenen bilgilerde eksiklik olup olmadığını kontrol et
                if any(value is None for value in company_info.__dict__.values()):
                    text_widget.insert(tk.END, f"{company_name}.txt dosyasındaki bilgiler eksik.\n\n")
                else:
                    company_text = (
                    f"Şirket Adı: {company_info.company_name}\n"
                    f"İletişim Bilgileri:\n"
                    f"Adres: {company_info.contact_info.address}\n"
                    f"Telefon Numarası: {company_info.contact_info.phone_number}\n"
                    f"E-posta Adresi: {company_info.contact_info.mail_address}\n"
                    f"Müşteri Bilgileri:\n"
                    f"Müşteri Adı: {company_info.customer_info.customer_name}\n"
                    f"Müşteri Soyadı: {company_info.customer_info.customer_surname}\n"
                    f"Müşteri Telefonu: {company_info.customer_info.customer_phone_number}\n"
                    f"Müşteri Adresi: {company_info.customer_info.customer_address}\n"
                    f"Çalışan Bilgileri:\n"
                    f"Çalışan Adı: {company_info.employee_info.employee_name}\n"
                    f"Çalışan Soyadı: {company_info.employee_info.employee_surname}\n"
                    f"Çalışan Maaşı: {company_info.employee_info.employee_salary}\n"
                    f"Mali Bilgiler:\n"
                    f"Gelir: {company_info.financial_info.income}\n"
                    f"Ödeme: {company_info.financial_info.payment}\n"
                    f"Satış: {company_info.financial_info.sales}\n\n"
                    )
                    text_widget.insert(tk.END, company_text)
            else:
                text_widget.insert(tk.END, f"{company_name}.txt dosyası yüklenemedi.\n\n")

        text_widget.config(state=tk.DISABLED)  # Text widget'ı sadece okunur yapmak için
    
    
if __name__ == "__main__":
    app = CompanyManagement()
    app.mainloop()