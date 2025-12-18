# kid-clothing-store-
#include <iostream>
#include <vector>
#include <string>
#include <sqlite3.h>
#include <iomanip>

// ==========================================
// 1. THE DATA MODEL (The structure of our data)
// ==========================================
struct Product {
    int id;
    std::string name;
    std::string age_range;
    double price;
    int stock;
};

// ==========================================
// 2. THE BACKEND (Database & Business Logic)
// ==========================================
class StoreBackend {
private:
    sqlite3* db;

public:
    StoreBackend(const std::string& db_name) {
        if (sqlite3_open(db_name.c_str(), &db) != SQLITE_OK) {
            std::cerr << "CRITICAL ERROR: Could not open database." << std::endl;
        }
        initialize_database();
    }

    ~StoreBackend() {
        sqlite3_close(db);
    }

    void initialize_database() {
        const char* sql = 
            "CREATE TABLE IF NOT EXISTS Inventory ("
            "id INTEGER PRIMARY KEY AUTOINCREMENT,"
            "name TEXT NOT NULL,"
            "age_range TEXT,"
            "price REAL,"
            "stock INTEGER);";
        sqlite3_exec(db, sql, 0, 0, 0);
    }

    // Backend Logic: Add new items to the store
    void addProduct(std::string name, std::string age, double price, int qty) {
        std::string sql = "INSERT INTO Inventory (name, age_range, price, stock) VALUES ('" 
                          + name + "', '" + age + "', " + std::to_string(price) + ", " + std::to_string(qty) + ");";
        sqlite3_exec(db, sql.c_str(), 0, 0, 0);
    }

    // Backend Logic: Handle a sale (reduces stock)
    bool processSale(int productID) {
        std::string sql = "UPDATE Inventory SET stock = stock - 1 WHERE id = " + std::to_string(productID) + " AND stock > 0;";
        int rc = sqlite3_exec(db, sql.c_str(), 0, 0, 0);
        return (rc == SQLITE_OK);
    }

    // Backend Logic: Fetch all data
    std::vector<Product> getAllProducts() {
        std::vector<Product> products;
        sqlite3_stmt* stmt;
        const char* sql = "SELECT * FROM Inventory;";

        if (sqlite3_prepare_v2(db, sql, -1, &stmt, 0) == SQLITE_OK) {
            while (sqlite3_step(stmt) == SQLITE_ROW) {
                products.push_back({
                    sqlite3_column_int(stmt, 0),
                    reinterpret_cast<const char*>(sqlite3_column_text(stmt, 1)),
                    reinterpret_cast<const char*>(sqlite3_column_text(stmt, 2)),
                    sqlite3_column_double(stmt, 3),
                    sqlite3_column_int(stmt, 4)
                });
            }
        }
        sqlite3_finalize(stmt);
        return products;
    }
};

// ==========================================
// 3. THE FRONTEND (User Interface Logic)
// ==========================================
class StoreUI {
private:
    StoreBackend& backend;

public:
    StoreUI(StoreBackend& b) : backend(b) {}

    void displayInventory() {
        std::vector<Product> items = backend.getAllProducts();
        
        std::cout << "\n" << std::string(60, '=') << "\n";
        std::cout << std::left << std::setw(5) << "ID" << std::setw(20) << "Item Name" 
                  << std::setw(15) << "Age Group" << std::setw(10) << "Price" << "Stock\n";
        std::cout << std::string(60, '-') << "\n";

        for (const auto& item : items) {
            std::cout << std::left << std::setw(5) << item.id 
                      << std::setw(20) << item.name 
                      << std::setw(15) << item.age_range 
                      << "$" << std::setw(9) << std::fixed << std::setprecision(2) << item.price 
                      << item.stock << (item.stock < 5 ? " (LOW STOCK)" : "") << "\n";
        }
        std::cout << std::string(60, '=') << "\n";
    }

    void showMenu() {
        int choice = 0;
        while (choice != 4) {
            std::cout << "\nKIDS CLOTHING STORE MANAGEMENT\n";
            std::cout << "1. View Inventory\n2. Sell Item\n3. Add New Stock\n4. Exit\nSelection: ";
            std::cin >> choice;

            if (choice == 1) {
                displayInventory();
            } else if (choice == 2) {
                int id;
                std::cout << "Enter Product ID to sell: ";
                std::cin >> id;
                if (backend.processSale(id)) std::cout << "Sale processed successfully!\n";
            } else if (choice == 3) {
                std::string n, a; double p; int s;
                std::cout << "Name: "; std::cin.ignore(); std::getline(std::cin, n);
                std::cout << "Age Group: "; std::getline(std::cin, a);
                std::cout << "Price: "; std::cin >> p;
                std::cout << "Quantity: "; std::cin >> s;
                backend.addProduct(n, a, p, s);
            }
        }
    }
};

// ==========================================
// 4. MAIN EXECUTION
// ==========================================
int main() {
    // Initialize Backend
    StoreBackend storeEngine("kids_shop.db");

    // Initialize Frontend and link to Backend
    StoreUI gui(storeEngine);

    // Run the application
    gui.showMenu();

    return 0;
}
