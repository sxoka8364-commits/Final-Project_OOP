// DentaCare - single-file version. Save as DentaCareApp.java, compile, and run DentaCareApp.
import java.awt.*;
import java.io.*;
import java.io.Serializable;
import java.time.*;
import java.time.LocalDate;
import java.time.LocalTime;
import java.time.format.DateTimeFormatter;
import java.util.ArrayList;
import java.util.Collections;
import java.util.Date;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.TreeMap;
import javax.swing.*;
import javax.swing.border.EmptyBorder;
import javax.swing.table.*;
import javax.swing.table.DefaultTableCellRenderer;
import javax.swing.table.DefaultTableModel;

// ===== DentaCareApp =====
/** Entry point: run this class. Opens the staff login window. */
public class DentaCareApp {
    public static void main(String[] args) {
        SwingUtilities.invokeLater(() -> {
            Ui.applyFonts();
            new LoginFrame(ClinicData.load()).setVisible(true);
        });
    }
}

// ===== LoginFrame =====
/** Staff login window. */
class LoginFrame extends JFrame {
    LoginFrame(ClinicData data) {
        super("DentaCare - Staff Login");
        setDefaultCloseOperation(EXIT_ON_CLOSE);
        JPanel root = new JPanel(new BorderLayout());
        JPanel banner = new JPanel(new GridLayout(2, 1));
        banner.setBackground(Ui.PRIMARY); banner.setBorder(new EmptyBorder(20, 24, 20, 24));
        JLabel t = new JLabel("DentaCare"); t.setFont(new Font("SansSerif", Font.BOLD, 30)); t.setForeground(Color.WHITE);
        JLabel s = new JLabel("Dental Clinic Management System"); s.setForeground(new Color(224, 242, 241));
        banner.add(t); banner.add(s);

        JPanel form = new JPanel(new GridBagLayout());
        form.setBorder(new EmptyBorder(16, 20, 8, 20));
        JComboBox<Staff> who = new JComboBox<>(data.getStaff().toArray(new Staff[0]));
        JPasswordField pin = new JPasswordField(12);
        Ui.addRow(form, 0, "Staff:", who);
        Ui.addRow(form, 1, "PIN:", pin);

        JButton login = Ui.button("Log In", Ui.PRIMARY);
        JLabel hint = new JLabel("Demo PINs: Maria (Receptionist) 1234, Ana (Assistant) 5678");
        hint.setForeground(Color.GRAY); hint.setFont(hint.getFont().deriveFont(11f));
        JPanel south = new JPanel(new BorderLayout(0, 8)); south.setBorder(new EmptyBorder(0, 20, 16, 20));
        south.add(login, BorderLayout.NORTH); south.add(hint, BorderLayout.SOUTH);

        Runnable doLogin = () -> {
            Staff st = (Staff) who.getSelectedItem();
            if (st != null && st.checkPin(new String(pin.getPassword()))) {
                new MainFrame(data, st).setVisible(true);
                dispose();
            } else { Ui.error(this, "Incorrect PIN. Please try again."); pin.setText(""); }
        };
        login.addActionListener(e -> doLogin.run());
        pin.addActionListener(e -> doLogin.run());

        root.add(banner, BorderLayout.NORTH); root.add(form, BorderLayout.CENTER); root.add(south, BorderLayout.SOUTH);
        setContentPane(root);
        pack(); setSize(Math.max(getWidth(), 420), getHeight()); setLocationRelativeTo(null); setResizable(false);
    }
}

// ===== MainFrame =====
class MainFrame extends JFrame {
    private final ClinicData data;
    private final Staff staff;
    private final JTabbedPane tabs = new JTabbedPane();

    // dashboard
    private final JPanel cardsPanel = new JPanel(new GridLayout(1, 4, 12, 0));
    private final DefaultTableModel apptModel = Ui.model("ID", "Patient", "Dentist", "Service", "Date", "Time", "Status");
    private final JTable apptTable = Ui.table(apptModel);
    private final JComboBox<String> statusFilter = new JComboBox<>();
    private final List<AppointmentProcess> shown = new ArrayList<>();

    // patients
    private final DefaultTableModel patientModel = Ui.model("ID", "Name", "Age", "Category", "Contact", "Allergy");
    private final JTable patientTable = Ui.table(patientModel);

    // booking
    private final JComboBox<Patient> bookPatient = new JComboBox<>();
    private final JComboBox<Dentist> bookDentist = new JComboBox<>();
    private final JComboBox<String> bookCategory = new JComboBox<>(new String[]{"Check-Up", "Procedure"});
    private final JComboBox<String> bookService = new JComboBox<>();
    private final JSpinner bookDate = Ui.dateSpinner();
    private final JComboBox<String> bookTime = new JComboBox<>(Ui.TIME_SLOTS);
    private final JLabel bookInfo = new JLabel(" ");

    // checkout
    private final JComboBox<AppointmentProcess> checkoutCombo = new JComboBox<>();
    private final JCheckBox discountBox = new JCheckBox("Senior Citizen / PWD discount (20%)");
    private final JTextArea billArea = new JTextArea();
    private CheckOut currentBill;

    public MainFrame(ClinicData data, Staff staff) {
        super("DentaCare - Dental Clinic Management System");
        this.data = data; this.staff = staff;
        setDefaultCloseOperation(EXIT_ON_CLOSE);
        setSize(1050, 680); setLocationRelativeTo(null);

        JPanel root = new JPanel(new BorderLayout());
        root.setBackground(Ui.BG);
        root.add(buildHeader(), BorderLayout.NORTH);
        tabs.addTab("Dashboard", buildDashboard());
        tabs.addTab("Patients", buildPatients());
        tabs.addTab("Book Appointment", buildBooking());
        tabs.addTab("Checkout", buildCheckout());
        tabs.addChangeListener(e -> refreshAll());
        root.add(tabs, BorderLayout.CENTER);
        setContentPane(root);
        refreshAll();
    }

    // ===================== HEADER =====================
    private JPanel buildHeader() {
        JPanel h = new JPanel(new BorderLayout());
        h.setBackground(Ui.PRIMARY); h.setBorder(new EmptyBorder(12, 20, 12, 20));
        JLabel t = new JLabel("DentaCare  |  Dental Clinic Management System");
        t.setForeground(Color.WHITE); t.setFont(new Font("SansSerif", Font.BOLD, 20));
        JLabel who = new JLabel("Logged in: " + staff + "   ");
        who.setForeground(Color.WHITE);
        JButton out = Ui.button("Log out", Ui.PRIMARY_DARK);
        out.addActionListener(e -> { dispose(); new LoginFrame(data).setVisible(true); });
        JPanel right = new JPanel(new FlowLayout(FlowLayout.RIGHT, 6, 0)); right.setOpaque(false);
        right.add(who); right.add(out);
        h.add(t, BorderLayout.WEST); h.add(right, BorderLayout.EAST);
        return h;
    }

    // ===================== DASHBOARD =====================
    private JPanel buildDashboard() {
        JPanel p = new JPanel(new BorderLayout(0, 12));
        p.setBorder(new EmptyBorder(14, 14, 14, 14)); p.setBackground(Ui.BG);
        cardsPanel.setOpaque(false);

        statusFilter.addItem("ALL");
        for (AppointmentProcess.Status s : AppointmentProcess.Status.values()) statusFilter.addItem(s.name());
        statusFilter.addActionListener(e -> refreshAppointments());

        JPanel top = new JPanel(new BorderLayout(0, 10)); top.setOpaque(false);
        JPanel bar = new JPanel(new FlowLayout(FlowLayout.LEFT, 8, 0)); bar.setOpaque(false);
        bar.add(Ui.title("Appointments")); bar.add(new JLabel("   Filter:")); bar.add(statusFilter);
        top.add(cardsPanel, BorderLayout.NORTH); top.add(bar, BorderLayout.SOUTH);

        apptTable.getColumnModel().getColumn(6).setCellRenderer(new StatusRenderer());

        JButton approve = Ui.button("Approve", Ui.PRIMARY);
        JButton reject = Ui.button("Reject", Ui.DANGER);
        JButton resched = Ui.button("Reschedule & Announce", new Color(2, 119, 189));
        JButton remind = Ui.button("Send Reminder", new Color(94, 53, 177));
        JButton record = Ui.button("Record Findings / Complete", new Color(239, 108, 0));
        approve.addActionListener(e -> approveSelected());
        reject.addActionListener(e -> rejectSelected());
        resched.addActionListener(e -> rescheduleSelected());
        remind.addActionListener(e -> remindSelected());
        record.addActionListener(e -> recordFindings());

        JPanel btns = new JPanel(new FlowLayout(FlowLayout.LEFT, 8, 4)); btns.setOpaque(false);
        btns.add(approve); btns.add(reject); btns.add(resched); btns.add(remind); btns.add(record);

        p.add(top, BorderLayout.NORTH);
        p.add(new JScrollPane(apptTable), BorderLayout.CENTER);
        p.add(btns, BorderLayout.SOUTH);
        return p;
    }

    private AppointmentProcess selectedAppointment() {
        int r = apptTable.getSelectedRow();
        if (r < 0) { Ui.error(this, "Please select an appointment from the table first."); return null; }
        return shown.get(r);
    }

    private void approveSelected() {
        AppointmentProcess a = selectedAppointment(); if (a == null) return;
        if (a.getStatus() != AppointmentProcess.Status.PENDING) { Ui.error(this, "Only PENDING appointments can be approved."); return; }
        a.setStatus(AppointmentProcess.Status.APPROVED);
        a.getService().approveDentalProcedure(a.getDentist().getFullName());
        a.notifyPatient("Your appointment " + a.getAppointmentId() + " was approved for " + a.getDateTimeText() + ".");
        data.save(); refreshAll();
        Ui.info(this, "Approved.\n\n" + a.getService().getApprovedDentalProcedure());
    }

    private void rejectSelected() {
        AppointmentProcess a = selectedAppointment(); if (a == null) return;
        if (a.getStatus() != AppointmentProcess.Status.PENDING) { Ui.error(this, "Only PENDING appointments can be rejected."); return; }
        String reason = JOptionPane.showInputDialog(this, "Reason for rejection:");
        if (reason == null) return;
        if (reason.trim().isEmpty()) { Ui.error(this, "Please enter a reason."); return; }
        a.setStatus(AppointmentProcess.Status.REJECTED);
        a.notifyPatient("Appointment " + a.getAppointmentId() + " was rejected: " + reason.trim());
        data.save(); refreshAll();
    }

    private void remindSelected() {
        AppointmentProcess a = selectedAppointment(); if (a == null) return;
        Ui.info(this, staff.remindAppointmentDate(a));
    }

    private void rescheduleSelected() {
        AppointmentProcess a = selectedAppointment(); if (a == null) return;
        if (a.getStatus() != AppointmentProcess.Status.PENDING && a.getStatus() != AppointmentProcess.Status.APPROVED) {
            Ui.error(this, "Only PENDING or APPROVED appointments can be rescheduled."); return;
        }
        JSpinner date = Ui.dateSpinner(); JComboBox<String> time = new JComboBox<>(Ui.TIME_SLOTS);
        JPanel form = new JPanel(new GridBagLayout());
        Ui.addRow(form, 0, "New date:", date); Ui.addRow(form, 1, "New time:", time);
        if (JOptionPane.showConfirmDialog(this, form, "Reschedule " + a.getAppointmentId(),
                JOptionPane.OK_CANCEL_OPTION, JOptionPane.PLAIN_MESSAGE) != JOptionPane.OK_OPTION) return;
        LocalDate d = Ui.toDate(date); LocalTime t = LocalTime.parse((String) time.getSelectedItem());
        if (d.isBefore(LocalDate.now())) { Ui.error(this, "The new date cannot be in the past."); return; }
        if (data.isSlotTaken(a.getDentist(), d, t, a)) { Ui.error(this, "That dentist is already booked at the chosen time."); return; }
        Ui.info(this, "Announcement sent:\n" + staff.announceUpdatedAppointmentDate(a, d, t));
        data.save(); refreshAll();
    }

    private void recordFindings() {
        AppointmentProcess a = selectedAppointment(); if (a == null) return;
        if (!a.getDentist().authorizeDentalRecordAccess(staff)) { denyAccess(); return; }
        if (a.getStatus() != AppointmentProcess.Status.APPROVED) { Ui.error(this, "Only APPROVED appointments can be completed."); return; }
        DentalService svc = a.getService();

        JSpinner tooth = new JSpinner(new SpinnerNumberModel(1, 1, 32, 1));
        JTextField cond = new JTextField(14);
        JTextArea chart = new JTextArea(4, 22); chart.setEditable(false);
        JButton add = Ui.button("Add to chart", Ui.PRIMARY);
        add.addActionListener(e -> {
            try { svc.updateTooth((Integer) tooth.getValue(), cond.getText()); cond.setText(""); chart.setText(chartText(svc)); }
            catch (IllegalArgumentException ex) { Ui.error(this, ex.getMessage()); }
        });
        JTextField findings = new JTextField(22), treatment = new JTextField(22), plan = new JTextField(22);

        JPanel f = new JPanel(new GridBagLayout());
        Ui.addRow(f, 0, "Procedure:", new JLabel("<html>" + svc.getDentalProcedure() + "</html>"));
        Ui.addRow(f, 1, "Tooth # (1-32):", tooth);
        Ui.addRow(f, 2, "Condition:", cond);
        Ui.addRow(f, 3, "", add);
        Ui.addRow(f, 4, "Dental chart:", new JScrollPane(chart));
        Ui.addRow(f, 5, "Findings:", findings);
        Ui.addRow(f, 6, "Treatment done:", treatment);
        Ui.addRow(f, 7, "Care plan:", plan);

        if (JOptionPane.showConfirmDialog(this, f, "Complete " + a.getAppointmentId(),
                JOptionPane.OK_CANCEL_OPTION, JOptionPane.PLAIN_MESSAGE) != JOptionPane.OK_OPTION) return;
        if (findings.getText().trim().isEmpty()) { Ui.error(this, "Findings are required to complete the visit."); return; }

        if (svc instanceof CheckUps) ((CheckUps) svc).setResults(findings.getText(), treatment.getText(), plan.getText());
        a.getPatient().addDentalRecord(a.getDateText() + " | " + svc.getServiceName() + " | " + a.getDentist().getFullName()
                + " | Findings: " + findings.getText().trim()
                + (treatment.getText().trim().isEmpty() ? "" : " | Treatment: " + treatment.getText().trim())
                + (plan.getText().trim().isEmpty() ? "" : " | Care plan: " + plan.getText().trim()));
        a.setStatus(AppointmentProcess.Status.COMPLETED);
        data.save(); refreshAll();
        Ui.info(this, "Visit completed and saved to the patient's dental record.\nProceed to the Checkout tab.");
    }

    private String chartText(DentalService svc) {
        StringBuilder sb = new StringBuilder();
        for (Map.Entry<Integer, String> e : svc.getDentalChart().entrySet())
            sb.append("Tooth #").append(e.getKey()).append(": ").append(e.getValue()).append('\n');
        return sb.toString();
    }

    private void denyAccess() {
        Ui.error(this, "Access denied.\nYour role (" + staff.getRole() + ") is not authorized to open dental records.");
    }

    // ===================== PATIENTS =====================
    private JPanel buildPatients() {
        JPanel p = new JPanel(new BorderLayout(14, 0));
        p.setBorder(new EmptyBorder(14, 14, 14, 14)); p.setBackground(Ui.BG);

        JTextField name = new JTextField(16), age = new JTextField(), contact = new JTextField(), guardian = new JTextField();
        JCheckBox allergy = new JCheckBox("Has allergy"); JTextField allergen = new JTextField(); allergen.setEnabled(false);
        JTextArea history = new JTextArea(3, 16); history.setLineWrap(true); history.setWrapStyleWord(true);
        allergy.addActionListener(e -> allergen.setEnabled(allergy.isSelected()));

        JPanel form = new JPanel(new GridBagLayout());
        form.setBorder(BorderFactory.createTitledBorder("Register new patient"));
        Ui.addRow(form, 0, "Full name:", name);
        Ui.addRow(form, 1, "Age:", age);
        Ui.addRow(form, 2, "Contact (09XXXXXXXXX):", contact);
        Ui.addRow(form, 3, "Guardian (if under 18):", guardian);
        Ui.addRow(form, 4, "", allergy);
        Ui.addRow(form, 5, "Allergic to:", allergen);
        Ui.addRow(form, 6, "Medical history:", new JScrollPane(history));
        JButton save = Ui.button("Register Patient", Ui.PRIMARY);
        save.addActionListener(e -> {
            try {
                Patient pt = data.registerPatient(name.getText(), age.getText(), contact.getText(), guardian.getText(),
                        allergy.isSelected(), allergen.getText(), history.getText());
                name.setText(""); age.setText(""); contact.setText(""); guardian.setText(""); allergen.setText("");
                allergy.setSelected(false); allergen.setEnabled(false); history.setText("");
                refreshAll();
                Ui.info(this, "Registered " + pt.getName() + " as " + pt.getPatientId() + ".");
            } catch (IllegalArgumentException ex) { Ui.error(this, ex.getMessage()); }
        });
        Ui.addRow(form, 7, "", save);

        JPanel left = new JPanel(new BorderLayout()); left.setOpaque(false); left.add(form, BorderLayout.NORTH);
        left.setPreferredSize(new Dimension(360, 100));

        JButton view = Ui.button("View Dental Record", new Color(2, 119, 189));
        view.addActionListener(e -> viewRecord());
        JPanel right = new JPanel(new BorderLayout(0, 8)); right.setOpaque(false);
        right.add(Ui.title("Patient List"), BorderLayout.NORTH);
        right.add(new JScrollPane(patientTable), BorderLayout.CENTER);
        JPanel b = new JPanel(new FlowLayout(FlowLayout.LEFT)); b.setOpaque(false); b.add(view);
        right.add(b, BorderLayout.SOUTH);

        p.add(left, BorderLayout.WEST); p.add(right, BorderLayout.CENTER);
        return p;
    }

    private void viewRecord() {
        int r = patientTable.getSelectedRow();
        if (r < 0) { Ui.error(this, "Select a patient first."); return; }
        if (!data.getDentists().get(0).authorizeDentalRecordAccess(staff)) { denyAccess(); return; }
        Patient p = data.getPatients().get(r);
        StringBuilder sb = new StringBuilder();
        sb.append("PATIENT: ").append(p).append("\nCategory: ").append(p.getPatientCategory())
          .append("\nAge: ").append(p.getAge()).append("   Contact: ").append(p.getContactNumber())
          .append("\nMedical history: ").append(p.getMedicalHistory())
          .append("\nAllergy: ").append(p.hasAllergy() ? p.getAllergen() : "None").append("\n\n--- DENTAL RECORDS ---\n");
        if (p.getDentalRecords().isEmpty()) sb.append("(no completed visits yet)\n");
        for (String rec : p.getDentalRecords()) sb.append("* ").append(rec).append('\n');
        sb.append("\n--- APPOINTMENT HISTORY ---\n");
        for (AppointmentProcess a : data.getAppointments())
            if (a.getPatient() == p) {
                sb.append(a.getAppointmentId()).append(" | ").append(a.getDateTimeText()).append(" | ")
                  .append(a.getService().getServiceName()).append(" | ").append(a.getStatus()).append('\n');
                String chart = chartText(a.getService());
                if (!chart.isEmpty()) sb.append(chart.replaceAll("(?m)^", "     "));
            }
        JTextArea area = new JTextArea(sb.toString(), 18, 55); area.setEditable(false); area.setLineWrap(true);
        JOptionPane.showMessageDialog(this, new JScrollPane(area), "Dental Record - " + p.getName(), JOptionPane.PLAIN_MESSAGE);
    }

    // ===================== BOOKING =====================
    private JPanel buildBooking() {
        JPanel p = new JPanel(new BorderLayout()); p.setBackground(Ui.BG); p.setBorder(new EmptyBorder(14, 14, 14, 14));
        JPanel form = new JPanel(new GridBagLayout());
        form.setBorder(BorderFactory.createCompoundBorder(BorderFactory.createTitledBorder("Appointment booking form"),
                new EmptyBorder(8, 8, 8, 8)));
        Ui.addRow(form, 0, "Patient:", bookPatient);
        Ui.addRow(form, 1, "Dentist:", bookDentist);
        Ui.addRow(form, 2, "Service type:", bookCategory);
        Ui.addRow(form, 3, "Service:", bookService);
        Ui.addRow(form, 4, "Date:", bookDate);
        Ui.addRow(form, 5, "Time:", bookTime);
        bookInfo.setForeground(Ui.PRIMARY_DARK);
        Ui.addRow(form, 6, "Details:", bookInfo);
        JButton book = Ui.button("Submit Appointment", Ui.PRIMARY);
        Ui.addRow(form, 7, "", book);

        bookCategory.addActionListener(e -> reloadServices());
        bookService.addActionListener(e -> updateBookInfo());
        bookPatient.addActionListener(e -> updateBookInfo());
        book.addActionListener(e -> {
            try {
                AppointmentProcess a = data.bookAppointment((Patient) bookPatient.getSelectedItem(),
                        (Dentist) bookDentist.getSelectedItem(), (String) bookCategory.getSelectedItem(),
                        (String) bookService.getSelectedItem(), Ui.toDate(bookDate),
                        LocalTime.parse((String) bookTime.getSelectedItem()));
                refreshAll();
                Ui.info(this, "Appointment " + a.getAppointmentId() + " submitted (status: PENDING).\n"
                        + a.getService().getDentalProcedure() + "\nWaiting for staff approval on the Dashboard.");
            } catch (IllegalArgumentException ex) { Ui.error(this, ex.getMessage()); }
        });
        JPanel wrap = new JPanel(new FlowLayout(FlowLayout.CENTER)); wrap.setOpaque(false);
        form.setPreferredSize(new Dimension(620, 380)); wrap.add(form);
        p.add(wrap, BorderLayout.CENTER);
        reloadServices();
        return p;
    }

    private void reloadServices() {
        bookService.removeAllItems();
        Map<String, Double> list = "Check-Up".equals(bookCategory.getSelectedItem()) ? CheckUps.PRICES : StandardProcedure.PRICES;
        for (String s : list.keySet()) bookService.addItem(s);
        updateBookInfo();
    }

    private void updateBookInfo() {
        String s = (String) bookService.getSelectedItem();
        Patient pt = (Patient) bookPatient.getSelectedItem();
        if (s == null) { bookInfo.setText(" "); return; }
        boolean check = "Check-Up".equals(bookCategory.getSelectedItem());
        double price = check ? CheckUps.PRICES.get(s) : StandardProcedure.PRICES.get(s);
        String txt = String.format("Price: PHP %.2f", price);
        if (!check && pt != null && pt.hasAllergy())
            txt += "  (Allergy: " + pt.getAllergen() + " - Modified Procedure, +PHP 500.00)";
        bookInfo.setText(txt);
    }

    // ===================== CHECKOUT =====================
    private JPanel buildCheckout() {
        JPanel p = new JPanel(new BorderLayout(0, 10)); p.setBackground(Ui.BG); p.setBorder(new EmptyBorder(14, 14, 14, 14));
        JButton gen = Ui.button("Generate Bill", Ui.PRIMARY);
        JButton pay = Ui.button("Confirm Payment", new Color(46, 125, 50));
        discountBox.setOpaque(false);
        JPanel top = new JPanel(new FlowLayout(FlowLayout.LEFT, 8, 4)); top.setOpaque(false);
        top.add(new JLabel("Completed visit:")); top.add(checkoutCombo); top.add(discountBox); top.add(gen); top.add(pay);
        billArea.setEditable(false); billArea.setFont(new Font(Font.MONOSPACED, Font.PLAIN, 14));
        billArea.setBorder(new EmptyBorder(10, 10, 10, 10));

        gen.addActionListener(e -> {
            AppointmentProcess a = (AppointmentProcess) checkoutCombo.getSelectedItem();
            if (a == null) { Ui.error(this, "There are no completed visits waiting for payment."); return; }
            currentBill = new CheckOut(a, discountBox.isSelected());
            billArea.setText(currentBill.buildReceipt());
        });
        pay.addActionListener(e -> {
            AppointmentProcess a = (AppointmentProcess) checkoutCombo.getSelectedItem();
            if (a == null || currentBill == null) { Ui.error(this, "Generate the bill first."); return; }
            a.setAmountPaid(currentBill.getTotal()); a.setStatus(AppointmentProcess.Status.PAID);
            billArea.append(String.format("%nPAID in full. Thank you, %s!%n", a.getPatient().getName()));
            currentBill = null; data.save(); refreshDashboardOnly();
            Ui.info(this, "Payment recorded.");
        });
        p.add(top, BorderLayout.NORTH); p.add(new JScrollPane(billArea), BorderLayout.CENTER);
        return p;
    }

    // ===================== REFRESH =====================
    private void refreshAll() {
        refreshDashboardOnly();
        patientModel.setRowCount(0);
        for (Patient pt : data.getPatients())
            patientModel.addRow(new Object[]{pt.getPatientId(), pt.getName(), pt.getAge(), pt.getPatientCategory(),
                    pt.getContactNumber(), pt.hasAllergy() ? pt.getAllergen() : "-"});

        Object selP = bookPatient.getSelectedItem(), selD = bookDentist.getSelectedItem();
        bookPatient.removeAllItems(); for (Patient pt : data.getPatients()) bookPatient.addItem(pt);
        bookDentist.removeAllItems(); for (Dentist d : data.getDentists()) bookDentist.addItem(d);
        if (selP != null) bookPatient.setSelectedItem(selP);
        if (selD != null) bookDentist.setSelectedItem(selD);

        checkoutCombo.removeAllItems();
        for (AppointmentProcess a : data.getAppointments())
            if (a.getStatus() == AppointmentProcess.Status.COMPLETED) checkoutCombo.addItem(a);
    }

    private void refreshDashboardOnly() {
        cardsPanel.removeAll();
        cardsPanel.add(Ui.card("Registered patients", String.valueOf(data.getPatients().size()), Ui.PRIMARY));
        cardsPanel.add(Ui.card("Pending approval", String.valueOf(data.countStatus(AppointmentProcess.Status.PENDING)), new Color(239, 108, 0)));
        cardsPanel.add(Ui.card("Approved", String.valueOf(data.countStatus(AppointmentProcess.Status.APPROVED)), new Color(2, 119, 189)));
        cardsPanel.add(Ui.card("Revenue collected", String.format("PHP %,.2f", data.totalRevenue()), new Color(46, 125, 50)));
        cardsPanel.revalidate(); cardsPanel.repaint();
        refreshAppointments();
    }

    private void refreshAppointments() {
        apptModel.setRowCount(0); shown.clear();
        String f = (String) statusFilter.getSelectedItem();
        for (AppointmentProcess a : data.getAppointments()) {
            if (f != null && !f.equals("ALL") && !a.getStatus().name().equals(f)) continue;
            shown.add(a);
            apptModel.addRow(new Object[]{a.getAppointmentId(), a.getPatient().getName(), a.getDentist().getFullName(),
                    a.getService().getServiceName(), a.getDateText(), a.getTimeText(), a.getStatus()});
        }
    }

    /** Colours the status column. */
    private static class StatusRenderer extends DefaultTableCellRenderer {
        @Override public Component getTableCellRendererComponent(JTable t, Object v, boolean sel, boolean foc, int r, int c) {
            Component comp = super.getTableCellRendererComponent(t, v, sel, foc, r, c);
            setFont(Ui.BOLD);
            if (!sel) {
                String s = String.valueOf(v);
                comp.setForeground(s.equals("PENDING") ? new Color(230, 81, 0) : s.equals("APPROVED") ? new Color(2, 119, 189)
                        : s.equals("REJECTED") ? Ui.DANGER : s.equals("COMPLETED") ? new Color(94, 53, 177) : new Color(46, 125, 50));
            }
            return comp;
        }
    }
}

// ===== Ui =====
/** Small helper class so every screen shares one look. */
final class Ui {
    public static final Color PRIMARY = new Color(0, 121, 107);
    public static final Color PRIMARY_DARK = new Color(0, 77, 64);
    public static final Color BG = new Color(240, 247, 246);
    public static final Color DANGER = new Color(198, 40, 40);
    public static final Font BASE = new Font("SansSerif", Font.PLAIN, 14);
    public static final Font BOLD = new Font("SansSerif", Font.BOLD, 14);
    public static final String[] TIME_SLOTS = {"09:00", "10:00", "11:00", "13:00", "14:00", "15:00", "16:00"};

    private Ui() { }

    public static void applyFonts() {
        for (Object key : UIManager.getDefaults().keySet())
            if (key.toString().endsWith(".font")) UIManager.put(key, BASE);
    }

    public static JButton button(String text, Color color) {
        JButton b = new JButton(text);
        b.setBackground(color); b.setForeground(Color.WHITE); b.setFont(BOLD);
        b.setOpaque(true); b.setBorderPainted(false); b.setFocusPainted(false);
        b.setBorder(new EmptyBorder(9, 16, 9, 16));
        b.setCursor(Cursor.getPredefinedCursor(Cursor.HAND_CURSOR));
        return b;
    }

    public static JLabel title(String text) {
        JLabel l = new JLabel(text);
        l.setFont(new Font("SansSerif", Font.BOLD, 18)); l.setForeground(PRIMARY_DARK);
        return l;
    }

    public static JPanel card(String heading, String value, Color color) {
        JPanel p = new JPanel(new GridLayout(2, 1));
        p.setBackground(color); p.setBorder(new EmptyBorder(12, 18, 12, 18));
        JLabel v = new JLabel(value); v.setFont(new Font("SansSerif", Font.BOLD, 26)); v.setForeground(Color.WHITE);
        JLabel h = new JLabel(heading); h.setForeground(new Color(230, 245, 243));
        p.add(v); p.add(h);
        return p;
    }

    public static JTable table(DefaultTableModel m) {
        JTable t = new JTable(m);
        t.setRowHeight(28); t.setFillsViewportHeight(true);
        t.setSelectionBackground(new Color(178, 223, 219)); t.setSelectionForeground(Color.BLACK);
        t.getTableHeader().setBackground(PRIMARY); t.getTableHeader().setForeground(Color.WHITE);
        t.getTableHeader().setFont(BOLD);
        t.setSelectionMode(ListSelectionModel.SINGLE_SELECTION);
        return t;
    }

    public static DefaultTableModel model(String... cols) {
        return new DefaultTableModel(cols, 0) {
            @Override public boolean isCellEditable(int r, int c) { return false; }
        };
    }

    public static void addRow(JPanel p, int row, String label, JComponent c) {
        GridBagConstraints g = new GridBagConstraints();
        g.insets = new Insets(6, 6, 6, 6); g.anchor = GridBagConstraints.NORTHWEST;
        g.gridx = 0; g.gridy = row; p.add(new JLabel(label), g);
        g.gridx = 1; g.fill = GridBagConstraints.HORIZONTAL; g.weightx = 1; p.add(c, g);
    }

    public static JSpinner dateSpinner() {
        JSpinner s = new JSpinner(new SpinnerDateModel(new Date(), null, null, java.util.Calendar.DAY_OF_MONTH));
        s.setEditor(new JSpinner.DateEditor(s, "MMM dd, yyyy"));
        return s;
    }

    public static LocalDate toDate(JSpinner s) {
        return ((Date) s.getValue()).toInstant().atZone(ZoneId.systemDefault()).toLocalDate();
    }

    public static void error(Component parent, String msg) {
        JOptionPane.showMessageDialog(parent, msg, "Please check", JOptionPane.WARNING_MESSAGE);
    }

    public static void info(Component parent, String msg) {
        JOptionPane.showMessageDialog(parent, msg, "DentaCare", JOptionPane.INFORMATION_MESSAGE);
    }
}

// ===== ClinicData =====
/**
 * ABSTRACTION: the GUI only calls simple methods (registerPatient, bookAppointment).
 * Validation, ID generation and file saving are hidden here.
 */
class ClinicData implements Serializable {
    private static final long serialVersionUID = 1L;
    private static final String FILE = "dentacare.dat";

    private final List<Patient> patients = new ArrayList<>();
    private final List<Dentist> dentists = new ArrayList<>();
    private final List<Staff> staff = new ArrayList<>();
    private final List<AppointmentProcess> appointments = new ArrayList<>();

    private ClinicData() { }

    public static ClinicData load() {
        File f = new File(FILE);
        if (f.exists()) {
            try (ObjectInputStream in = new ObjectInputStream(new FileInputStream(f))) {
                return (ClinicData) in.readObject();
            } catch (Exception e) { System.out.println("Could not read saved data; starting fresh."); }
        }
        ClinicData d = new ClinicData();
        d.seed();
        return d;
    }

    public void save() {
        try (ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream(FILE))) {
            out.writeObject(this);
        } catch (IOException e) { System.out.println("Save failed: " + e.getMessage()); }
    }

    private void seed() {
        staff.add(new Staff("EMP-001", "Maria Santos", "Receptionist", "1234"));
        staff.add(new Staff("EMP-002", "Ana Cruz", "Assistant", "5678"));
        dentists.add(new Dentist("Dr. Rafael Dela Cruz", "DMD", "General Dentistry", "PRC-0012345"));
        dentists.add(new Dentist("Dr. Lourdes Reyes", "DDM, MS Orthodontics", "Orthodontics", "PRC-0023456"));
        dentists.add(new Dentist("Dr. Mark Tan", "DMD, Oral Surgery Fellow", "Oral Surgery", "PRC-0034567"));
        patients.add(new AdultPatient("PT-001", "Juan Dela Pena", 34, "09171234567", "Hypertension", true, "Penicillin"));
        patients.add(new MinorPatient("PT-002", "Mia Lopez", 9, "09281234567", "Carmen Lopez", "None", false, ""));
        save();
    }

    // ---------- Patients ----------
    public Patient registerPatient(String name, String ageText, String contact, String guardian,
                                   boolean allergy, String allergen, String history) {
        name = name.trim();
        if (!name.matches("[A-Za-z][A-Za-z .'-]{1,59}")) throw new IllegalArgumentException("Enter a valid full name (letters only).");
        int age;
        try { age = Integer.parseInt(ageText.trim()); }
        catch (NumberFormatException e) { throw new IllegalArgumentException("Age must be a whole number."); }
        if (age < 0 || age > 120) throw new IllegalArgumentException("Age must be between 0 and 120.");
        if (!contact.trim().matches("(09|\\+639)\\d{9}")) throw new IllegalArgumentException("Contact number must look like 09XXXXXXXXX.");
        if (allergy && allergen.trim().isEmpty()) throw new IllegalArgumentException("Please state what the patient is allergic to.");
        if (age < 18 && guardian.trim().isEmpty()) throw new IllegalArgumentException("A guardian's name is required for patients under 18.");
        String hist = history.trim().isEmpty() ? "None" : history.trim();
        String id = String.format("PT-%03d", patients.size() + 1);
        Patient p = age < 18
                ? new MinorPatient(id, name, age, contact.trim(), guardian.trim(), hist, allergy, allergen.trim())
                : new AdultPatient(id, name, age, contact.trim(), hist, allergy, allergen.trim());
        patients.add(p);
        save();
        return p;
    }

    // ---------- Appointments ----------
    public AppointmentProcess bookAppointment(Patient p, Dentist d, String category, String serviceName,
                                              LocalDate date, LocalTime time) {
        if (p == null) throw new IllegalArgumentException("Please select a patient.");
        if (d == null) throw new IllegalArgumentException("Please select a dentist.");
        if (serviceName == null) throw new IllegalArgumentException("Please select a service.");
        if (date.isBefore(LocalDate.now())) throw new IllegalArgumentException("The appointment date cannot be in the past.");
        if (isSlotTaken(d, date, time, null))
            throw new IllegalArgumentException(d.getFullName() + " is already booked at that date and time.");

        DentalService service;
        if (category.equals("Check-Up")) service = new CheckUps(serviceName);
        else if (p.hasAllergy()) service = new ModifiedProcedure(serviceName, p.getAllergen());   // POLYMORPHISM
        else service = new StandardProcedure(serviceName, d.getFullName());

        String id = String.format("APT-%03d", appointments.size() + 1);
        AppointmentProcess ap = d.setAppointment(id, p, service, date, time);
        appointments.add(ap);
        save();
        return ap;
    }

    public boolean isSlotTaken(Dentist d, LocalDate date, LocalTime time, AppointmentProcess ignore) {
        for (AppointmentProcess a : appointments) {
            if (a == ignore) continue;
            boolean active = a.getStatus() == AppointmentProcess.Status.PENDING
                    || a.getStatus() == AppointmentProcess.Status.APPROVED;
            if (active && a.getDentist() == d && a.getAppointedDate().equals(date) && a.getAppointedTime().equals(time))
                return true;
        }
        return false;
    }

    public double totalRevenue() {
        return appointments.stream().mapToDouble(AppointmentProcess::getAmountPaid).sum();
    }

    public long countStatus(AppointmentProcess.Status s) {
        return appointments.stream().filter(a -> a.getStatus() == s).count();
    }

    public List<Patient> getPatients() { return patients; }
    public List<Dentist> getDentists() { return dentists; }
    public List<Staff> getStaff() { return staff; }
    public List<AppointmentProcess> getAppointments() { return appointments; }
}

// ===== Patient =====
/** ABSTRACT + ENCAPSULATION: all details are private, accessed via getters/setters. */
abstract class Patient implements Serializable {
    private static final long serialVersionUID = 1L;

    private final String patientId;
    private String name;
    private int age;
    private String contactNumber;
    private String medicalHistory;
    private boolean hasAllergy;
    private String allergen;
    private final List<String> dentalRecords = new ArrayList<>();

    protected Patient(String patientId, String name, int age, String contactNumber,
                      String medicalHistory, boolean hasAllergy, String allergen) {
        this.patientId = patientId;
        this.name = name;
        this.age = age;
        this.contactNumber = contactNumber;
        this.medicalHistory = medicalHistory;
        this.hasAllergy = hasAllergy;
        this.allergen = allergen;
    }

    /** Polymorphic: each patient type describes itself differently. */
    public abstract String getPatientCategory();

    public String getPatientId() { return patientId; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public int getAge() { return age; }
    public void setAge(int age) { this.age = age; }
    public String getContactNumber() { return contactNumber; }
    public void setContactNumber(String c) { this.contactNumber = c; }
    public String getMedicalHistory() { return medicalHistory; }
    public void setMedicalHistory(String m) { this.medicalHistory = m; }
    public boolean hasAllergy() { return hasAllergy; }
    public String getAllergen() { return allergen; }
    public void setAllergy(boolean has, String allergen) { this.hasAllergy = has; this.allergen = allergen; }

    public List<String> getDentalRecords() { return Collections.unmodifiableList(dentalRecords); }
    public void addDentalRecord(String record) { dentalRecords.add(record); }

    @Override public String toString() { return patientId + " - " + name; }
}

// ===== AdultPatient =====
/** INHERITANCE: adult patient (18+). */
class AdultPatient extends Patient {
    private static final long serialVersionUID = 1L;

    public AdultPatient(String id, String name, int age, String contact,
                        String history, boolean allergy, String allergen) {
        super(id, name, age, contact, history, allergy, allergen);
    }

    @Override public String getPatientCategory() { return "Adult"; }
}

// ===== MinorPatient =====
/** INHERITANCE: minor patient who must have a guardian on record. */
class MinorPatient extends Patient {
    private static final long serialVersionUID = 1L;
    private String guardianName;

    public MinorPatient(String id, String name, int age, String contact, String guardianName,
                        String history, boolean allergy, String allergen) {
        super(id, name, age, contact, history, allergy, allergen);
        this.guardianName = guardianName;
    }

    public String getGuardianName() { return guardianName; }
    public void setGuardianName(String g) { this.guardianName = g; }

    @Override public String getPatientCategory() { return "Minor (Guardian: " + guardianName + ")"; }
}

// ===== DentalService =====
/** ABSTRACT BASE CLASS for every dental service (INHERITANCE + POLYMORPHISM). */
abstract class DentalService implements Price, Serializable {
    private static final long serialVersionUID = 1L;

    private final String serviceName;
    private final Map<Integer, String> dentalChart = new TreeMap<>();   // tooth no. -> condition
    private String approvedDentalProcedure = "Awaiting approval";

    protected DentalService(String serviceName) { this.serviceName = serviceName; }

    /** Overridden by each subclass: describes how the procedure will be done. */
    public abstract String getDentalProcedure();
    public abstract String getCategory();

    public void approveDentalProcedure(String dentistName) {
        approvedDentalProcedure = getDentalProcedure() + "  [approved by " + dentistName + "]";
    }

    public void updateTooth(int tooth, String condition) {
        if (tooth < 1 || tooth > 32) throw new IllegalArgumentException("Tooth number must be 1 to 32.");
        if (condition == null || condition.trim().isEmpty())
            throw new IllegalArgumentException("Please enter the tooth condition.");
        dentalChart.put(tooth, condition.trim());
    }

    public String getServiceName() { return serviceName; }
    public String getApprovedDentalProcedure() { return approvedDentalProcedure; }
    public Map<Integer, String> getDentalChart() { return Collections.unmodifiableMap(dentalChart); }
    public double getPrice() { return getPriceList().getOrDefault(serviceName, 0.0); }
}

// ===== CheckUps =====
/** Check-up service (extends DentalService implements Price). */
class CheckUps extends DentalService implements Price {
    private static final long serialVersionUID = 1L;
    public static final Map<String, Double> PRICES = new LinkedHashMap<>();
    static {
        PRICES.put("Consultation", 300.0);
        PRICES.put("Full Oral Examination", 700.0);
        PRICES.put("Dental X-Ray", 500.0);
    }

    private String findings = "";
    private String treatments = "";
    private String carePlans = "";

    public CheckUps(String serviceName) { super(serviceName); }

    public void setResults(String findings, String treatments, String carePlans) {
        this.findings = findings; this.treatments = treatments; this.carePlans = carePlans;
    }

    public String getCheckUpResults() {
        return "Findings: " + findings + "\nTreatments: " + treatments + "\nCare plan: " + carePlans;
    }

    @Override public String getDentalProcedure() { return "Check-up: " + getServiceName() + " (examination only)"; }
    @Override public String getCategory() { return "Check-Up"; }
    @Override public Map<String, Double> getPriceList() { return new LinkedHashMap<>(PRICES); }
}

// ===== StandardProcedure =====
/** Standard procedure for patients without allergies. */
class StandardProcedure extends DentalService {
    private static final long serialVersionUID = 1L;
    public static final Map<String, Double> PRICES = new LinkedHashMap<>();
    static {
        PRICES.put("Oral Prophylaxis (Cleaning)", 800.0);
        PRICES.put("Tooth Extraction", 1000.0);
        PRICES.put("Tooth Filling", 1200.0);
        PRICES.put("Fluoride Treatment", 600.0);
        PRICES.put("Teeth Whitening", 5000.0);
    }

    private final String dentistName;
    private final boolean isQualified;

    public StandardProcedure(String serviceName, String dentistName) {
        super(serviceName);
        this.dentistName = dentistName;
        this.isQualified = dentistName != null && !dentistName.isEmpty();
    }

    public String toothExtraction(int tooth) {
        updateTooth(tooth, "Extracted");
        return "Tooth #" + tooth + " extracted by " + dentistName;
    }

    @Override public String getDentalProcedure() {
        return "Standard procedure: " + getServiceName() + " by " + dentistName
                + (isQualified ? " (qualified)" : "");
    }
    @Override public String getCategory() { return "Standard Procedure"; }
    @Override public Map<String, Double> getPriceList() { return new LinkedHashMap<>(PRICES); }
}

// ===== ModifiedProcedure =====
/** Modified procedure: used automatically when the patient has an allergy. */
class ModifiedProcedure extends DentalService {
    private static final long serialVersionUID = 1L;

    private final String isAllergicTo;
    private final List<String> alternatives = new ArrayList<>();

    public ModifiedProcedure(String serviceName, String allergen) {
        super(serviceName);
        this.isAllergicTo = allergen;
        alternatives.add(alteration());
    }

    /** Picks a safe alternative for the patient's allergen. */
    public String alteration() {
        String a = isAllergicTo == null ? "" : isAllergicTo.toLowerCase();
        if (a.contains("latex"))      return "Use latex-free gloves and rubber dam";
        if (a.contains("lidocaine") || a.contains("anesthe"))
                                      return "Use an alternative local anesthetic (e.g., articaine)";
        if (a.contains("penicillin")) return "Prescribe a penicillin-free antibiotic (e.g., azithromycin)";
        if (a.contains("nickel"))     return "Use nickel-free materials and instruments";
        return "Dentist to choose a hypoallergenic alternative for " + isAllergicTo;
    }

    public String getIsAllergicTo() { return isAllergicTo; }
    public List<String> getAlternatives() { return alternatives; }

    @Override public String getDentalProcedure() {
        return "Modified procedure: " + getServiceName() + " | Allergy: " + isAllergicTo
                + " | Alternative: " + alternatives.get(0);
    }
    @Override public String getCategory() { return "Modified Procedure"; }
    @Override public Map<String, Double> getPriceList() { return new LinkedHashMap<>(StandardProcedure.PRICES); }
}

// ===== Price =====
/** Interface: anything that can publish a price list (used by services). */
interface Price {
    Map<String, Double> getPriceList();
}

// ===== DentalCosts =====
/** ABSTRACT: shared cost data; subclasses decide how fees are computed. */
abstract class DentalCosts {
    private final Map<String, Double> dentalServicePriceList = new LinkedHashMap<>(CheckUps.PRICES);
    private final Map<String, Double> dentalProcedurePriceList = new LinkedHashMap<>(StandardProcedure.PRICES);
    private final Map<String, Double> modificationFees = new LinkedHashMap<>();
    private final Map<String, Double> serviceFees = new LinkedHashMap<>();

    protected DentalCosts() {
        modificationFees.put("Allergy modification fee", 500.0);
        serviceFees.put("Clinic service fee", 100.0);
    }

    public abstract double calculateServiceFees();

    public Map<String, Double> getDentalServicePriceList() { return dentalServicePriceList; }
    public Map<String, Double> getDentalProcedurePriceList() { return dentalProcedurePriceList; }
    public Map<String, Double> getModificationFees() { return modificationFees; }
    public Map<String, Double> getServiceFees() { return serviceFees; }
}

// ===== CheckOut =====
/** Billing for a completed appointment (extends DentalCosts). */
class CheckOut extends DentalCosts {
    public static final double DISCOUNT_RATE = 0.20;   // Senior / PWD discount
    private final AppointmentProcess appointment;
    private final boolean hasDiscount;

    public CheckOut(AppointmentProcess appointment, boolean hasDiscount) {
        this.appointment = appointment;
        this.hasDiscount = hasDiscount;
    }

    private double base() { return appointment.getService().getPrice(); }
    private double modification() {
        return appointment.getService() instanceof ModifiedProcedure
                ? getModificationFees().get("Allergy modification fee") : 0.0;
    }
    private double discount() { return hasDiscount ? (base() + modification()) * DISCOUNT_RATE : 0.0; }
    private double clinicFees() { return getServiceFees().values().stream().mapToDouble(Double::doubleValue).sum(); }

    @Override public double calculateServiceFees() { return base() + modification() - discount() + clinicFees(); }

    public double getTotal() { return calculateServiceFees(); }

    public String buildReceipt() {
        StringBuilder sb = new StringBuilder();
        sb.append("==========  DENTACARE OFFICIAL BILL  ==========\n");
        sb.append("Appointment : ").append(appointment.getAppointmentId()).append('\n');
        sb.append("Patient     : ").append(appointment.getPatient().getName()).append('\n');
        sb.append("Dentist     : ").append(appointment.getDentist().getFullName()).append('\n');
        sb.append("Date        : ").append(appointment.getDateTimeText()).append("\n\n");
        sb.append(String.format("%-34s PHP %9.2f%n", appointment.getService().getServiceName(), base()));
        if (modification() > 0) sb.append(String.format("%-34s PHP %9.2f%n", "Allergy modification fee", modification()));
        sb.append(String.format("%-34s PHP %9.2f%n", "Clinic service fee", clinicFees()));
        if (hasDiscount) sb.append(String.format("%-34s PHP %9.2f%n", "Senior/PWD discount (20%)", -discount()));
        sb.append("--------------------------------------------\n");
        sb.append(String.format("%-34s PHP %9.2f%n", "TOTAL AMOUNT DUE", getTotal()));
        return sb.toString();
    }
}

// ===== Dentist =====
class Dentist implements Serializable {
    private static final long serialVersionUID = 1L;
    private final String fullName;
    private final String clinicalQualifications;
    private final String specialization;
    private final String licenseNumber;

    public Dentist(String fullName, String clinicalQualifications, String specialization, String licenseNumber) {
        this.fullName = fullName; this.clinicalQualifications = clinicalQualifications;
        this.specialization = specialization; this.licenseNumber = licenseNumber;
    }

    public AppointmentProcess setAppointment(String id, Patient p, DentalService s, LocalDate d, LocalTime t) {
        return new AppointmentProcess(id, p, this, s, d, t);
    }

    /** Only Receptionists may open patient records; others are denied (privacy objective). */
    public boolean authorizeDentalRecordAccess(Staff staff) {
        return staff != null && staff.getRole().equals("Receptionist");
    }

    public String getFullName() { return fullName; }
    public String getClinicalQualifications() { return clinicalQualifications; }
    public String getSpecialization() { return specialization; }
    public String getLicenseNumber() { return licenseNumber; }
    @Override public String toString() { return fullName + " - " + specialization; }
}

// ===== Staff =====
class Staff implements Serializable {
    private static final long serialVersionUID = 1L;
    private final String employeeId;
    private final String name;
    private final String role;
    private final String pin;

    public Staff(String employeeId, String name, String role, String pin) {
        this.employeeId = employeeId; this.name = name; this.role = role; this.pin = pin;
    }

    public boolean checkPin(String attempt) { return pin.equals(attempt); }
    public String getEmployeeId() { return employeeId; }
    public String getName() { return name; }
    public String getRole() { return role; }

    public String remindAppointmentDate(AppointmentProcess ap) { return ap.remindPatient(); }

    public String announceUpdatedAppointmentDate(AppointmentProcess ap, LocalDate d, LocalTime t) {
        ap.setAppointment(d, t);
        String msg = "Appointment " + ap.getAppointmentId() + " was moved to " + ap.getDateTimeText() + ".";
        ap.notifyPatient(msg);
        return msg;
    }

    @Override public String toString() { return name + " (" + role + ")"; }
}

// ===== AppointmentProcess =====
class AppointmentProcess implements Serializable {
    private static final long serialVersionUID = 1L;
    public enum Status { PENDING, APPROVED, REJECTED, COMPLETED, PAID }
    private static final DateTimeFormatter DATE_FMT = DateTimeFormatter.ofPattern("MMM dd, yyyy");
    private static final DateTimeFormatter TIME_FMT = DateTimeFormatter.ofPattern("hh:mm a");

    private final String appointmentId;
    private final Patient patient;
    private final Dentist dentist;
    private final DentalService service;
    private LocalDate appointedDate;
    private LocalTime appointedTime;
    private Status status = Status.PENDING;
    private double amountPaid;
    private final List<String> notifications = new ArrayList<>();

    public AppointmentProcess(String id, Patient p, Dentist d, DentalService s, LocalDate date, LocalTime time) {
        this.appointmentId = id; this.patient = p; this.dentist = d; this.service = s;
        this.appointedDate = date; this.appointedTime = time;
    }

    public void setAppointment(LocalDate d, LocalTime t) { this.appointedDate = d; this.appointedTime = t; }

    public String remindPatient() {
        return "Reminder for " + patient.getName() + " (" + patient.getContactNumber() + "): "
                + service.getServiceName() + " with " + dentist.getFullName() + " on " + getDateTimeText() + ".";
    }

    public void notifyPatient(String message) { notifications.add(message); }

    public String getDateTimeText() { return appointedDate.format(DATE_FMT) + " at " + appointedTime.format(TIME_FMT); }
    public String getDateText() { return appointedDate.format(DATE_FMT); }
    public String getTimeText() { return appointedTime.format(TIME_FMT); }

    public String getAppointmentId() { return appointmentId; }
    public Patient getPatient() { return patient; }
    public Dentist getDentist() { return dentist; }
    public DentalService getService() { return service; }
    public LocalDate getAppointedDate() { return appointedDate; }
    public LocalTime getAppointedTime() { return appointedTime; }
    public Status getStatus() { return status; }
    public void setStatus(Status s) { this.status = s; }
    public double getAmountPaid() { return amountPaid; }
    public void setAmountPaid(double a) { this.amountPaid = a; }
    public List<String> getNotifications() { return notifications; }

    @Override public String toString() {
        return appointmentId + " | " + patient.getName() + " | " + service.getServiceName();
    }
}
