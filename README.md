// RealEstateSystem.java
import java.time.LocalDate;
import java.time.format.DateTimeFormatter;
import java.time.temporal.ChronoUnit;
import java.util.*;

/**
 * RealEstateSystem - Menu-driven console application for Real Estate Leasing & Rent Collection
 *
 * Compile:
 *   javac RealEstateSystem.java
 * Run:
 *   java RealEstateSystem
 */
public class RealEstateSystem {
    private static final Scanner scanner = new Scanner(System.in);
    private static final DateTimeFormatter DF = DateTimeFormatter.ofPattern("yyyy-MM-dd");

    // Data stores
    private final Map<Integer, Property> properties = new LinkedHashMap<>();
    private final Map<Integer, Tenant> tenants = new LinkedHashMap<>();
    private final Map<Integer, Lease> leases = new LinkedHashMap<>();
    private final Map<Integer, RentInvoice> invoices = new LinkedHashMap<>();
    private final Map<Integer, Payment> payments = new LinkedHashMap<>();
    private final Map<Integer, MaintenanceRequest> maintenanceRequests = new LinkedHashMap<>();

    // Auto-increment counters
    private int propertyIdSeq = 1;
    private int unitIdSeq = 1;
    private int tenantIdSeq = 1;
    private int leaseIdSeq = 1;
    private int invoiceIdSeq = 1;
    private int paymentIdSeq = 1;
    private int maintenanceIdSeq = 1;

    public static void main(String[] args) {
        RealEstateSystem app = new RealEstateSystem();
        app.bootstrapSampleData(); // optional: sample data to try
        app.runMenu();
    }

    private void runMenu() {
        while (true) {
            System.out.println("\n--- Real Estate Leasing & Rent Collection ---");
            System.out.println("1. Add Property");
            System.out.println("2. Add Unit");
            System.out.println("3. Add Tenant");
            System.out.println("4. Create Lease");
            System.out.println("5. Generate Invoice");
            System.out.println("6. Record Payment");
            System.out.println("7. Log Maintenance");
            System.out.println("8. Display Units / Occupancy");
            System.out.println("9. Show Lease Summary");
            System.out.println("10. Show Invoice Details");
            System.out.println("11. Show Maintenance Log");
            System.out.println("0. Exit");
            System.out.print("Choose: ");
            String choice = scanner.nextLine().trim();
            try {
                switch (choice) {
                    case "1": cmdAddProperty(); break;
                    case "2": cmdAddUnit(); break;
                    case "3": cmdAddTenant(); break;
                    case "4": cmdCreateLease(); break;
                    case "5": cmdGenerateInvoice(); break;
                    case "6": cmdRecordPayment(); break;
                    case "7": cmdLogMaintenance(); break;
                    case "8": cmdDisplayUnits(); break;
                    case "9": cmdShowLeaseSummary(); break;
                    case "10": cmdShowInvoiceDetails(); break;
                    case "11": cmdShowMaintenanceLog(); break;
                    case "0": System.out.println("Exiting..."); return;
                    default: System.out.println("Invalid choice. Try again."); break;
                }
            } catch (Exception ex) {
                System.out.println("Error: " + ex.getMessage());
            }
        }
    }

    // ---------- Commands ----------

    private void cmdAddProperty() {
        System.out.print("Property name: ");
        String name = readNonEmpty();
        System.out.print("Address: ");
        String address = readNonEmpty();
        Property p = new Property(propertyIdSeq++, name, address);
        properties.put(p.getId(), p);
        System.out.println("Property added: " + p);
    }

    private void cmdAddUnit() {
        if (properties.isEmpty()) {
            System.out.println("No properties exist. Add property first.");
            return;
        }
        System.out.println("Choose property by id:");
        properties.values().forEach(prop -> System.out.println(prop.getId() + ": " + prop.getName()));
        int pid = readInt("Property id: ");
        Property p = properties.get(pid);
        if (p == null) {
            System.out.println("Property not found.");
            return;
        }
        System.out.print("Unit number (like 101A): ");
        String unitNo = readNonEmpty();
        double rent = readDouble("Monthly rent amount: ");
        Unit u = new Unit(unitIdSeq++, unitNo, rent);
        p.addUnit(u);
        System.out.println("Unit added to property: " + u);
    }

    private void cmdAddTenant() {
        System.out.print("Tenant name: ");
        String name = readNonEmpty();
        System.out.print("Email: ");
        String email = readEmail();
        System.out.print("Phone: ");
        String phone = readNonEmpty();
        Tenant t = new Tenant(tenantIdSeq++, name, email, phone);
        tenants.put(t.getId(), t);
        System.out.println("Tenant added: " + t);
    }

    private void cmdCreateLease() {
        if (properties.isEmpty()) {
            System.out.println("No properties/units exist.");
            return;
        }
        // List vacant units
        List<Unit> vacantUnits = new ArrayList<>();
        System.out.println("Vacant Units:");
        for (Property p : properties.values()) {
            for (Unit u : p.getUnits()) {
                if (!u.isOccupied()) {
                    vacantUnits.add(u);
                    System.out.println(u.getId() + ": " + p.getName() + " - Unit " + u.getUnitNumber() + " (Rent: " + u.getRent() + ")");
                }
            }
        }
        if (vacantUnits.isEmpty()) {
            System.out.println("No vacant units available.");
            return;
        }
        int uid = readInt("Choose unit id: ");
        Unit chosen = null;
        for (Unit u : vacantUnits) if (u.getId() == uid) chosen = u;
        if (chosen == null) {
            System.out.println("Invalid unit selected.");
            return;
        }
        if (tenants.isEmpty()) {
            System.out.println("No tenants exist. Add tenant first.");
            return;
        }
        System.out.println("Tenants:");
        tenants.values().forEach(t -> System.out.println(t.getId() + ": " + t.getName() + " (" + t.getEmail() + ")"));
        int tid = readInt("Choose tenant id: ");
        Tenant tenant = tenants.get(tid);
        if (tenant == null) {
            System.out.println("Tenant not found.");
            return;
        }
        LocalDate start = readDate("Lease start date (yyyy-MM-dd): ");
        LocalDate end = readDate("Lease end date (yyyy-MM-dd): ");
        if (end.isBefore(start)) {
            System.out.println("End date must be after start date.");
            return;
        }
        double rent = chosen.getRent();
        int cycleDays = 30; // monthly by default
        Lease lease = new Lease(leaseIdSeq++, chosen, tenant, start, end, rent, cycleDays);
        // Business rule: lease activation updates unit occupancy immediately
        lease.activate();
        leases.put(lease.getId(), lease);
        System.out.println("Lease created and activated: " + lease.brief());
    }

    private void cmdGenerateInvoice() {
        if (leases.isEmpty()) {
            System.out.println("No leases available.");
            return;
        }
        System.out.println("Active leases:");
        leases.values().forEach(l -> System.out.println(l.getId() + ": " + l.brief()));
        int lid = readInt("Choose lease id to generate invoice for: ");
        Lease lease = leases.get(lid);
        if (lease == null) {
            System.out.println("Lease not found.");
            return;
        }
        LocalDate dueDate = readDate("Invoice due date (yyyy-MM-dd): ");
        // Business rule: invoices per lease cycle before payment
        double base = lease.getRent();
        RentInvoice inv = new RentInvoice(invoiceIdSeq++, lease, dueDate, base);
        invoices.put(inv.getId(), inv);
        // compute penalty if dueDate < today and unpaid
        if (LocalDate.now().isAfter(dueDate)) {
            long daysLate = ChronoUnit.DAYS.between(dueDate, LocalDate.now());
            inv.applyLateFee(daysLate);
        }
        System.out.println("Invoice generated:\n" + inv.fullString());
    }

    private void cmdRecordPayment() {
        if (invoices.isEmpty()) {
            System.out.println("No invoices present. Generate invoice first.");
            return;
        }
        System.out.println("Unpaid invoices:");
        invoices.values().stream().filter(inv -> !inv.isPaid()).forEach(inv -> System.out.println(inv.getId() + ": Lease " + inv.getLease().getId() + " Due: " + inv.getDueDate() + " Amount: " + inv.getTotalAmount()));
        int iid = readInt("Invoice id to pay: ");
        RentInvoice inv = invoices.get(iid);
        if (inv == null) {
            System.out.println("Invoice not found.");
            return;
        }
        if (inv.isPaid()) {
            System.out.println("Invoice already fully paid.");
            return;
        }
        System.out.println("Choose payment type: 1. Cash  2. Card");
        String pty = scanner.nextLine().trim();
        Payment payment;
        double amt = readDouble("Payment amount: ");
        LocalDate payDate = LocalDate.now();
        if ("2".equals(pty)) {
            System.out.print("Card last 4 digits: ");
            String last4 = scanner.nextLine().trim();
            payment = new CardPayment(paymentIdSeq++, payDate, amt, "CARD", last4);
        } else {
            payment = new CashPayment(paymentIdSeq++, payDate, amt, "CASH");
        }
        payments.put(payment.getId(), payment);
        inv.applyPayment(payment);
        System.out.println("Payment recorded. Invoice status: " + (inv.isPaid() ? "PAID" : "PARTIAL"));
    }

    private void cmdLogMaintenance() {
        if (properties.isEmpty()) {
            System.out.println("No properties/units exist.");
            return;
        }
        System.out.println("Units:");
        properties.values().forEach(prop -> {
            for (Unit u : prop.getUnits()) {
                System.out.println(u.getId() + ": " + prop.getName() + " - " + u.getUnitNumber() + " (Occupied: " + u.isOccupied() + ")");
            }
        });
        int uid = readInt("Unit id: ");
        Unit u = findUnitById(uid);
        if (u == null) {
            System.out.println("Unit not found.");
            return;
        }
        System.out.println("Tenants:");
        tenants.values().forEach(t -> System.out.println(t.getId() + ": " + t.getName()));
        int tid = readInt("Tenant id (must be occupant): ");
        Tenant tenant = tenants.get(tid);
        if (tenant == null) {
            System.out.println("Tenant not found.");
            return;
        }
        if (!u.isOccupied() || !isTenantOccupyingUnit(tenant, u)) {
            System.out.println("Tenant is not the occupant of this unit.");
            return;
        }
        System.out.print("Describe maintenance issue: ");
        String desc = readNonEmpty();
        MaintenanceRequest mr = new MaintenanceRequest(maintenanceIdSeq++, u, tenant, desc);
        maintenanceRequests.put(mr.getId(), mr);
        System.out.println("Maintenance logged: " + mr.brief());
    }

    private void cmdDisplayUnits() {
        if (properties.isEmpty()) {
            System.out.println("No properties exist.");
            return;
        }
        for (Property p : properties.values()) {
            System.out.println("\nProperty: " + p.getName() + " (ID " + p.getId() + ")");
            for (Unit u : p.getUnits()) {
                System.out.println("  UnitId:" + u.getId() + " #" + u.getUnitNumber() + " Rent:" + u.getRent() + " Occupied:" + u.isOccupied());
            }
        }
    }

    private void cmdShowLeaseSummary() {
        if (leases.isEmpty()) {
            System.out.println("No leases.");
            return;
        }
        for (Lease l : leases.values()) {
            System.out.println(l.detailedString());
        }
    }

    private void cmdShowInvoiceDetails() {
        if (invoices.isEmpty()) {
            System.out.println("No invoices.");
            return;
        }
        for (RentInvoice inv : invoices.values()) {
            System.out.println(inv.fullString());
        }
    }

    private void cmdShowMaintenanceLog() {
        if (maintenanceRequests.isEmpty()) {
            System.out.println("No maintenance logs.");
            return;
        }
        for (MaintenanceRequest mr : maintenanceRequests.values()) {
            System.out.println(mr.fullString());
        }
    }

    // ---------- Helpers ----------

    private Unit findUnitById(int uid) {
        for (Property p : properties.values()) {
            for (Unit u : p.getUnits()) if (u.getId() == uid) return u;
        }
        return null;
    }

    private boolean isTenantOccupyingUnit(Tenant tenant, Unit unit) {
        for (Lease l : leases.values()) {
            if (l.getUnit().equals(unit) && l.getTenant().equals(tenant) && l.isActive()) {
                return true;
            }
        }
        return false;
    }

    private String readNonEmpty() {
        while (true) {
            String s = scanner.nextLine().trim();
            if (!s.isEmpty()) return s;
            System.out.print("Value cannot be empty. Enter again: ");
        }
    }

    private int readInt(String prompt) {
        System.out.print(prompt);
        while (true) {
            String line = scanner.nextLine().trim();
            try {
                return Integer.parseInt(line);
            } catch (NumberFormatException ex) {
                System.out.print("Enter a valid integer: ");
            }
        }
    }

    private double readDouble(String prompt) {
        System.out.print(prompt);
        while (true) {
            String line = scanner.nextLine().trim();
            try {
                double d = Double.parseDouble(line);
                if (d < 0) { System.out.print("Enter non-negative number: "); continue; }
                return d;
            } catch (NumberFormatException ex) {
                System.out.print("Enter a valid number: ");
            }
        }
    }

    private LocalDate readDate(String prompt) {
        System.out.print(prompt);
        while (true) {
            String line = scanner.nextLine().trim();
            try {
                return LocalDate.parse(line, DF);
            } catch (Exception ex) {
                System.out.print("Invalid date. Format yyyy-MM-dd. Try again: ");
            }
        }
    }

    private String readEmail() {
        while (true) {
            String e = scanner.nextLine().trim();
            if (e.matches("^[^@\\s]+@[^@\\s]+\\.[^@\\s]+$")) return e;
            System.out.print("Enter a valid email: ");
        }
    }

    // ---------- Sample bootstrap data ----------
    private void bootstrapSampleData() {
        Property p1 = new Property(propertyIdSeq++, "Sunrise Apartments", "12 East St");
        Property p2 = new Property(propertyIdSeq++, "Maple Residency", "89 North Rd");
        properties.put(p1.getId(), p1);
        properties.put(p2.getId(), p2);
        Unit u1 = new Unit(unitIdSeq++, "101A", 12000);
        Unit u2 = new Unit(unitIdSeq++, "102B", 14000);
        Unit u3 = new Unit(unitIdSeq++, "201", 10000);
        p1.addUnit(u1); p1.addUnit(u2); p2.addUnit(u3);

        Tenant t1 = new Tenant(tenantIdSeq++, "Akhil Kumar", "akhil@example.com", "9876543210");
        Tenant t2 = new Tenant(tenantIdSeq++, "Nisha Rao", "nisha@example.com", "9123456780");
        tenants.put(t1.getId(), t1);
        tenants.put(t2.getId(), t2);

        // Create a lease and activate it
        Lease l1 = new Lease(leaseIdSeq++, u1, t1, LocalDate.now().minusMonths(1), LocalDate.now().plusMonths(11), u1.getRent(), 30);
        l1.activate();
        leases.put(l1.getId(), l1);

        // Create one invoice (past due) to demonstrate penalties
        RentInvoice inv = new RentInvoice(invoiceIdSeq++, l1, LocalDate.now().minusDays(10), l1.getRent());
        inv.applyLateFee(10);
        invoices.put(inv.getId(), inv);
    }

}

/* -----------------------------
   MODELS: Property, Unit, Person/Tenant,
   Lease, RentInvoice, Payment, MaintenanceRequest
   ----------------------------- */

class Property {
    private final int id;
    private String name;
    private String address;
    private final List<Unit> units = new ArrayList<>();

    public Property(int id, String name, String address) {
        this.id = id; this.name = name; this.address = address;
    }

    public void addUnit(Unit u) { units.add(u); }

    public List<Unit> getUnits() { return Collections.unmodifiableList(units); }

    public int getId() { return id; }
    public String getName() { return name; }
    public String getAddress() { return address; }

    @Override
    public String toString() {
        return String.format("Property[%d] %s (%s) units=%d", id, name, address, units.size());
    }
}

class Unit {
    private final int id;
    private final String unitNumber;
    private double rent;
    private boolean occupied = false;

    public Unit(int id, String unitNumber, double rent) {
        this.id = id; this.unitNumber = unitNumber; this.rent = rent;
    }

    public int getId() { return id; }
    public String getUnitNumber() { return unitNumber; }
    public double getRent() { return rent; }
    public boolean isOccupied() { return occupied; }

    public void setOccupied(boolean val) { occupied = val; }

    @Override
    public String toString() {
        return String.format("Unit[%d] %s rent=%.2f occupied=%s", id, unitNumber, rent, occupied);
    }
}

/* Person -> Tenant (inheritance) */
abstract class Person {
    private String name;
    private String email;
    public Person(String name, String email) { this.name = name; this.email = email; }
    public String getName() { return name; }
    public String getEmail() { return email; }
}

class Tenant extends Person {
    private final int id;
    private String phone;

    public Tenant(int id, String name, String email, String phone) {
        super(name, email);
        this.id = id; this.phone = phone;
    }

    public int getId() { return id; }
    public String getPhone() { return phone; }

    @Override
    public String toString() {
        return String.format("Tenant[%d] %s (%s) phone=%s", id, getName(), getEmail(), phone);
    }
}

/* Lease */
class Lease {
    private final int id;
    private final Unit unit;
    private final Tenant tenant;
    private final LocalDate startDate;
    private final LocalDate endDate;
    private final double rent;
    private final int cycleDays;
    private boolean active = false;

    public Lease(int id, Unit unit, Tenant tenant, LocalDate start, LocalDate end, double rent, int cycleDays) {
        this.id = id; this.unit = unit; this.tenant = tenant; this.startDate = start; this.endDate = end; this.rent = rent; this.cycleDays = cycleDays;
    }

    public int getId() { return id; }
    public Unit getUnit() { return unit; }
    public Tenant getTenant() { return tenant; }
    public LocalDate getStartDate() { return startDate; }
    public LocalDate getEndDate() { return endDate; }
    public double getRent() { return rent; }
    public boolean isActive() { return active; }

    public void activate() {
        if (unit.isOccupied()) {
            throw new IllegalStateException("Unit already occupied. Cannot activate lease.");
        }
        active = true;
        unit.setOccupied(true); // business rule: occupancy updated immediately
    }

    public void terminate() {
        active = false;
        unit.setOccupied(false);
    }

    public String brief() {
        return String.format("Lease[%d] Unit:%s Tenant:%s Period:%s->%s Rent:%.2f Active:%s", id, unit.getUnitNumber(), tenant.getName(), startDate, endDate, rent, active);
    }

    public String detailedString() {
        return String.format("Lease ID: %d\n  Unit: %s (id:%d)\n  Tenant: %s (id:%d)\n  Start: %s\n  End:   %s\n  Rent: %.2f\n  Active: %s\n",
                id, unit.getUnitNumber(), unit.getId(), tenant.getName(), tenant.getId(), startDate, endDate, rent, active);
    }

}

/* RentInvoice */
class RentInvoice {
    private final int id;
    private final Lease lease;
    private final LocalDate dueDate;
    private final double baseAmount;
    private double penalty = 0.0;
    private final List<Payment> payments = new ArrayList<>();

    public RentInvoice(int id, Lease lease, LocalDate dueDate, double baseAmount) {
        this.id = id; this.lease = lease; this.dueDate = dueDate; this.baseAmount = baseAmount;
    }

    public int getId() { return id; }
    public Lease getLease() { return lease; }
    public LocalDate getDueDate() { return dueDate; }
    public double getBaseAmount() { return baseAmount; }

    public double getTotalAmount() {
        return baseAmount + penalty;
    }

    public boolean isPaid() {
        double paid = payments.stream().mapToDouble(Payment::getAmount).sum();
        return paid >= getTotalAmount() - 0.0001;
    }

    public void applyPayment(Payment p) {
        payments.add(p);
    }

    public void applyLateFee(long daysLate) {
        // Simple policy: 1% of base per day late (capped at 50% of base)
        double fee = baseAmount * 0.01 * daysLate;
        double cap = baseAmount * 0.5;
        this.penalty = Math.min(fee, cap);
    }

    public double getPenalty() { return penalty; }

    public String fullString() {
        StringBuilder sb = new StringBuilder();
        sb.append(String.format("Invoice[%d] Lease:%d Due:%s\n", id, lease.getId(), dueDate));
        sb.append(String.format("  Base: %.2f\n  Penalty: %.2f\n  Total: %.2f\n", baseAmount, penalty, getTotalAmount()));
        double paid = payments.stream().mapToDouble(Payment::getAmount).sum();
        sb.append(String.format("  Paid: %.2f  Status: %s\n", paid, isPaid() ? "PAID" : "UNPAID/PARTIAL"));
        if (!payments.isEmpty()) {
            sb.append("  Payments:\n");
            for (Payment p : payments) sb.append("    - ").append(p).append("\n");
        }
        return sb.toString();
    }
}

/* Payment (polymorphism + inheritance) */
abstract class Payment {
    private final int id;
    private final LocalDate date;
    private final double amount;
    private final String method;

    public Payment(int id, LocalDate date, double amount, String method) {
        this.id = id; this.date = date; this.amount = amount; this.method = method;
    }

    public int getId() { return id; }
    public LocalDate getDate() { return date; }
    public double getAmount() { return amount; }
    public String getMethod() { return method; }

    @Override
    public String toString() {
        return String.format("Payment[%d] %s %.2f on %s", id, method, amount, date);
    }
}

class CashPayment extends Payment {
    public CashPayment(int id, LocalDate date, double amount, String method) {
        super(id, date, amount, method);
    }
}

class CardPayment extends Payment {
    private final String last4;
    public CardPayment(int id, LocalDate date, double amount, String method, String last4) {
        super(id, date, amount, method);
        this.last4 = last4;
    }

    @Override
    public String toString() {
        return String.format("CardPayment[%d] %s %.2f on %s (card ****%s)", getId(), getMethod(), getAmount(), getDate(), last4);
    }
}

/* MaintenanceRequest */
enum MaintenanceStatus { LOGGED, ASSIGNED, IN_PROGRESS, RESOLVED, CLOSED }

class MaintenanceRequest {
    private final int id;
    private final Unit unit;
    private final Tenant tenant;
    private final LocalDate createdOn;
    private final String description;
    private MaintenanceStatus status = MaintenanceStatus.LOGGED;
    private String notes = "";

    public MaintenanceRequest(int id, Unit unit, Tenant tenant, String description) {
        this.id = id; this.unit = unit; this.tenant = tenant; this.description = description; this.createdOn = LocalDate.now();
    }

    public int getId() { return id; }
    public Unit getUnit() { return unit; }
    public Tenant getTenant() { return tenant; }
    public MaintenanceStatus getStatus() { return status; }
    public void setStatus(MaintenanceStatus s) { status = s; }
    public String getDescription() { return description; }

    public String brief() {
        return String.format("MR[%d] Unit:%s Tenant:%s Status:%s", id, unit.getUnitNumber(), tenant.getName(), status);
    }

    public String fullString() {
        return String.format("Maintenance[%d] Unit:%s (id:%d)\n  Tenant: %s (id:%d)\n  Created: %s\n  Desc: %s\n  Status: %s\n  Notes: %s\n",
                id, unit.getUnitNumber(), unit.getId(), tenant.getName(), tenant.getId(), createdOn, description, status, notes);
    }
}
