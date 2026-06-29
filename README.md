# monimport SwiftUI
import SwiftData

// MARK: - 1. MODÈLES DE DONNÉES ENRICHI (SwiftData)
@Model
class Appointment {
    var id: UUID
    var title: String
    var date: Date
    var colorHex: String
    var location: String
    var notes: String
    var iconName: String
    
    init(title: String, date: Date, colorHex: String = "#FF3B30", location: String = "", notes: String = "", iconName: String = "calendar") {
        self.id = UUID()
        self.title = title
        self.date = date
        self.colorHex = colorHex
        self.location = location
        self.notes = notes
        self.iconName = iconName
    }
}

@Model
class CustomColor {
    var id: UUID
    var name: String
    var hex: String
    
    init(name: String, hex: String) {
        self.id = UUID()
        self.name = name
        self.hex = hex
    }
}

@Model
class CalendarSettings {
    var id: UUID
    var bgHex: String
    var textHex: String
    var gridHex: String
    var weekendBgHex: String
    var holidayBgHex: String
    
    init(bgHex: String = "#F2F2F7", textHex: String = "#000000", gridHex: String = "#E5E5EA", weekendBgHex: String = "#FFE5E5", holidayBgHex: String = "#E5FFE5") {
        self.id = UUID()
        self.bgHex = bgHex
        self.textHex = textHex
        self.gridHex = gridHex
        self.weekendBgHex = weekendBgHex
        self.holidayBgHex = holidayBgHex
    }
}

// MARK: - 2. LANCEMENT DE L'APP
@main
struct RDVApp: App {
    var body: some Scene {
        WindowGroup {
            MainCalendarView()
        }
        .modelContainer(for: [Appointment.self, CustomColor.self, CalendarSettings.self])
    }
}

// MARK: - 3. INTERFACE PRINCIPALE
struct MainCalendarView: View {
    @Environment(\.modelContext) private var modelContext
    @Query private var allAppointments: [Appointment]
    @Query private var customColors: [CustomColor]
    @Query private var appSettings: [CalendarSettings]
    
    @State private var currentDate = Date()
    @State private var selectedDate = Date()
    @State private var isShowingForm = false
    @State private var isShowingSettings = false
    @State private var appointmentToEdit: Appointment?
    
    private var currentSettings: CalendarSettings {
        appSettings.first ?? CalendarSettings()
    }
    
    private var monthFormatter: DateFormatter {
        let formatter = DateFormatter()
        formatter.locale = Locale(identifier: "fr_FR")
        formatter.setLocalizedDateFormatFromTemplate("MMMM")
        return formatter
    }
    
    private var yearFormatter: DateFormatter {
        let formatter = DateFormatter()
        formatter.locale = Locale(identifier: "fr_FR")
        formatter.setLocalizedDateFormatFromTemplate("yyyy")
        return formatter
    }
    
    private var isCurrentMonth: Bool {
        Calendar.current.isDate(currentDate, equalTo: Date(), toGranularity: .month)
    }
    
    private var yearRange: [Int] {
        let currentYear = Calendar.current.component(.year, from: currentDate)
        return Array((currentYear - 10)...(currentYear + 20))
    }
    
    private var numberOfDaysInMonth: Int {
        Calendar.current.range(of: .day, in: .month, for: currentDate)?.count ?? 31
    }
    
    private var firstDayOffset: Int {
        let calendar = Calendar.current
        var components = calendar.dateComponents([.year, .month], from: currentDate)
        components.day = 1
        let firstDayDate = calendar.date(from: components) ?? currentDate
        let weekday = calendar.component(.weekday, from: firstDayDate)
        let offset = weekday - 2
        return offset < 0 ? 6 : offset
    }
    
    private var totalGridCells: Int {
        firstDayOffset + numberOfDaysInMonth
    }

    var body: some View {
        VStack(spacing: 0) {
            
            // BARRE DU HAUT GÉANTE
            HStack(spacing: 40) {
                Button(action: { isShowingSettings = true }) {
                    Image(systemName: "gearshape.fill")
                        .font(.largeTitle)
                        .foregroundColor(Color(hex: currentSettings.textHex))
                        .padding(20)
                        .background(Color(hex: currentSettings.textHex).opacity(0.08))
                        .clipShape(Circle())
                }
                .buttonStyle(.plain)
                
                Menu {
                    ScrollView {
                        ForEach(yearRange, id: \.self) { year in
                            Button(String(year)) { changeYear(to: year) }
                        }
                    }
                } label: {
                    HStack(spacing: 12) {
                        Text(yearFormatter.string(from: currentDate)).font(.title).bold()
                        Image(systemName: "chevron.down").font(.title2.bold())
                    }
                    .foregroundColor(.blue)
                    .padding(.vertical, 16).padding(.horizontal, 30)
                    .background(Color.blue.opacity(0.12)).cornerRadius(16)
                }
                .menuStyle(.button)
                
                Spacer()
                
                HStack(spacing: 30) {
                    Button(action: { changeMonth(by: -1) }) {
                        Image(systemName: "chevron.left").font(.largeTitle.bold())
                            .foregroundColor(Color(hex: currentSettings.textHex)).padding(20)
                            .background(Color(hex: currentSettings.textHex).opacity(0.05)).clipShape(Circle())
                    }
                    .buttonStyle(.plain)
                    
                    Menu {
                        Button("Mois actuel") {
                            withAnimation { currentDate = Date(); selectedDate = Date() }
                        }
                    } label: {
                        Text(monthFormatter.string(from: currentDate).capitalized)
                            .font(.system(size: 38, weight: .black))
                            .foregroundColor(Color(hex: currentSettings.textHex))
                            .frame(minWidth: 240)
                    }
                    .menuStyle(.button)
                    
                    Button(action: { changeMonth(by: 1) }) {
                        Image(systemName: "chevron.right").font(.largeTitle.bold())
                            .foregroundColor(Color(hex: currentSettings.textHex)).padding(20)
                            .background(Color(hex: currentSettings.textHex).opacity(0.05)).clipShape(Circle())
                    }
                    .buttonStyle(.plain)
                }
                
                Spacer()
                
                if !isCurrentMonth {
                    Button(action: {
                        withAnimation { currentDate = Date(); selectedDate = Date() }
                    }) {
                        Text("Aujourd'hui").font(.title2).bold().padding(.vertical, 18).padding(.horizontal, 35)
                            .background(Color.blue).foregroundColor(.white).cornerRadius(16)
                    }
                    .buttonStyle(.plain)
                } else {
                    Spacer().frame(width: 160)
                }
            }
            .padding(.horizontal, 35).padding(.vertical, 25)
            .background(Color(hex: currentSettings.gridHex).opacity(0.4))
            
            // CORPS DU CALENDRIER
            VStack {
                HStack {
                    ForEach(["Lun", "Mar", "Mer", "Jeu", "Ven", "Sam", "Dim"], id: \.self) { day in
                        Text(day)
                            .font(.title2.bold())
                            .foregroundColor(Color(hex: currentSettings.textHex))
                            .frame(maxWidth: .infinity)
                    }
                }
                .padding(.horizontal).padding(.top, 25)
                
                Divider().padding(.vertical, 15)
                
                GeometryReader { geometry in
                    let columns = Array(repeating: GridItem(.flexible(), spacing: 15), count: 7)
                    let rowCount = CGFloat(ceil(Double(totalGridCells) / 7.0))
                    let rowHeight = (geometry.size.height - (15 * (rowCount - 1))) / rowCount
                    
                    ScrollView(showsIndicators: false) {
                        LazyVGrid(columns: columns, spacing: 15) {
                            ForEach(0..<totalGridCells, id: \.self) { index in
                                
                                if index < firstDayOffset {
                                    Color.clear
                                        .frame(height: rowHeight)
                                } else {
                                    let day = index - firstDayOffset + 1
                                    let targetDate = dateFromDay(day)
                                    
                                    let rdvDuJour = appointmentsForDate(targetDate).sorted { $0.date < $1.date }
                                    
                                    let bgHexColor: String = {
                                        if isHoliday(targetDate) { return currentSettings.holidayBgHex }
                                        else if isWeekend(targetDate) { return currentSettings.weekendBgHex }
                                        else { return currentSettings.bgHex }
                                    }()
                                    
                                    VStack(spacing: 0) {
                                        HStack {
                                            Text("\(day)")
                                                .font(.system(size: 24, weight: .black, design: .rounded))
                                                .foregroundColor(Calendar.current.isDate(targetDate, inSameDayAs: Date()) ? .blue : Color(hex: bgHexColor).accessibleTextColor)
                                                .padding(14)
                                            Spacer()
                                        }
                                        
                                        // MODIFICATION ICI : Retour au rangement HORIZONTAL (`HStack`) avec pastilles XXL défilantes si ça déborde
                                        if !rdvDuJour.isEmpty {
                                            ScrollView(.horizontal, showsIndicators: false) {
                                                HStack(spacing: 12) {
                                                    ForEach(rdvDuJour) { rdv in
                                                        Image(systemName: rdv.iconName)
                                                            .font(.title3.bold()) // Icône interne plus grande (20pt)
                                                            .foregroundColor(rdv.colorHex.uppercased() == "#FFFFFF" ? .black : .white)
                                                            .frame(width: 42, height: 42) // Pastille XXL
                                                            .background(Color(hex: rdv.colorHex))
                                                            .clipShape(Circle())
                                                            .overlay(
                                                                Circle().stroke(rdv.colorHex.uppercased() == "#FFFFFF" ? Color.black : Color.white, lineWidth: 2.5)
                                                            )
                                                            .onTapGesture {
                                                                appointmentToEdit = rdv
                                                                selectedDate = rdv.date
                                                                isShowingForm = true
                                                            }
                                                    }
                                                }
                                                .padding(.horizontal, 14)
                                            }
                                        }
                                        Spacer()
                                    }
                                    .frame(height: rowHeight)
                                    .frame(maxWidth: .infinity)
                                    .background(Color(hex: bgHexColor))
                                    .cornerRadius(14)
                                    .overlay(
                                        RoundedRectangle(cornerRadius: 14)
                                            .stroke(Color(hex: currentSettings.gridHex), lineWidth: 2)
                                    )
                                    .contentShape(Rectangle())
                                    .onTapGesture {
                                        appointmentToEdit = nil
                                        selectedDate = targetDate
                                        isShowingForm = true
                                    }
                                }
                            }
                        }
                    }
                }
                .padding(.horizontal).padding(.bottom, 25)
            }
            .background(Color(.systemBackground))
        }
        .onAppear { populateDefaultsIfNeeded() }
        .sheet(isPresented: $isShowingForm) {
            AppointmentFormView(selectedDate: selectedDate, appointmentToEdit: appointmentToEdit, availableColors: customColors)
        }
        .sheet(isPresented: $isShowingSettings) {
            SettingsView(settings: currentSettings)
        }
    }
    
    private func isWeekend(_ date: Date) -> Bool {
        let weekday = Calendar.current.component(.weekday, from: date)
        return weekday == 7 || weekday == 1
    }
    
    private func isHoliday(_ date: Date) -> Bool {
        let calendar = Calendar.current
        let year = calendar.component(.year, from: date)
        let month = calendar.component(.month, from: date)
        let day = calendar.component(.day, from: date)
        
        if (month == 1 && day == 1) || (month == 5 && day == 1) || (month == 5 && day == 8) ||
           (month == 7 && day == 14) || (month == 8 && day == 15) || (month == 11 && day == 1) ||
           (month == 11 && day == 11) || (month == 12 && day == 25) {
            return true
        }
        
        let a = year % 19, b = year / 100, c = year % 100
        let d = b / 4, e = b % 4, f = (b + 8) / 25
        let g = (b - f + 1) / 3
        let h = (19 * a + b - d - g + 15) % 30
        let i = c / 4, k = c % 4
        let l = (32 + 2 * e + 2 * i - h - k) % 7
        let m = (a + 11 * h + 22 * l) / 451
        let easterMonth = (h + l - 7 * m + 114) / 31
        let easterDay = ((h + l - 7 * m + 114) % 31) + 1
        
        var comp = DateComponents()
        comp.year = year
        comp.month = easterMonth
        comp.day = easterDay
        
        guard let easterDate = calendar.date(from: comp) else { return false }
        let easterMonday = calendar.date(byAdding: .day, value: 1, to: easterDate)!
        let ascension = calendar.date(byAdding: .day, value: 39, to: easterDate)!
        let pentecostMonday = calendar.date(byAdding: .day, value: 50, to: easterDate)!
        
        return calendar.isDate(date, inSameDayAs: easterMonday) ||
               calendar.isDate(date, inSameDayAs: ascension) ||
               calendar.isDate(date, inSameDayAs: pentecostMonday)
    }
    
    private func changeMonth(by value: Int) {
        if let newDate = Calendar.current.date(byAdding: .month, value: value, to: currentDate) {
            withAnimation { currentDate = newDate }
        }
    }
    
    private func changeYear(to newYear: Int) {
        var components = Calendar.current.dateComponents([.month, .day], from: currentDate)
        components.year = newYear
        if let newDate = Calendar.current.date(from: components) {
            withAnimation { currentDate = newDate }
        }
    }
    
    private func dateFromDay(_ day: Int) -> Date {
        var components = Calendar.current.dateComponents([.year, .month], from: currentDate)
        components.day = day
        return Calendar.current.date(from: components) ?? currentDate
    }
    
    private func appointmentsForDate(_ date: Date) -> [Appointment] {
        allAppointments.filter { Calendar.current.isDate($0.date, inSameDayAs: date) }
    }
    
    private func populateDefaultsIfNeeded() {
        if customColors.isEmpty {
            let defaults = [
                CustomColor(name: "Travail", hex: "#007AFF"),
                CustomColor(name: "Perso", hex: "#34C759"),
                CustomColor(name: "Urgent", hex: "#FF3B30"),
                CustomColor(name: "Important", hex: "#000000"),
                CustomColor(name: "Blanc", hex: "#FFFFFF")
            ]
            for color in defaults { modelContext.insert(color) }
        }
        if appSettings.isEmpty {
            let defaultSettings = CalendarSettings(
                bgHex: "#F2F2F7",
                textHex: "#000000",
                gridHex: "#E5E5EA",
                weekendBgHex: "#FFE5E5",
                holidayBgHex: "#E5FFE5"
            )
            modelContext.insert(defaultSettings)
        }
    }
}

// MARK: - 4. VUE RÉGLAGES
struct SettingsView: View {
    @Environment(\.modelContext) private var modelContext
    @Environment(\.dismiss) private var dismiss
    @Query private var customColors: [CustomColor]
    
    @Bindable var settings: CalendarSettings
    
    @State private var newColorName = ""
    @State private var newColorHex = "#007AFF"
    
    let palette = [
        "#000000", "#3A3A3C", "#8E8E93", "#AEAEB2", "#E5E5EA", "#FFFFFF",
        "#007AFF", "#34C759", "#FF3B30", "#AF52DE", "#FF9500", "#FFCC00",
        "#4A2C11", "#A0522D", "#D2B48C", "#FFE5E5", "#E5FFE5", "#E5F5FF"
    ]

    var body: some View {
        NavigationStack {
            HStack(alignment: .top, spacing: 0) {
                Form {
                    Section(header: Text("Ajouter une catégorie").font(.title3.bold())) {
                        TextField("Nom (ex: Réunion, Privé...)", text: $newColorName).font(.title3).padding(.vertical, 4)
                        LazyVGrid(columns: Array(repeating: GridItem(.fixed(40), spacing: 10), count: 6), spacing: 10) {
                            ForEach(palette, id: \.self) { hex in
                                Circle()
                                    .fill(Color(hex: hex))
                                    .frame(width: 40, height: 40)
                                    .overlay(
                                        Circle().stroke(
                                            hex == "#FFFFFF" ? Color.black.opacity(0.3) : (hex == "#000000" ? Color.gray : Color.primary),
                                            lineWidth: newColorHex == hex ? 3 : (hex == "#FFFFFF" ? 1 : 0)
                                        )
                                    )
                                    .onTapGesture { newColorHex = hex }
                            }
                        }
                        .padding(.vertical, 8)
                        Button(action: addColor) { HStack { Spacer(); Text("Ajouter").font(.title3).bold(); Spacer() } }
                            .disabled(newColorName.isEmpty).buttonStyle(.borderedProminent)
                    }
                    
                    Section(header: Text("Noms des catégories").font(.title3.bold())) {
                        ForEach(customColors) { color in
                            @Bindable var bindableColor = color
                            HStack(spacing: 15) {
                                Circle().fill(Color(hex: color.hex)).frame(width: 30, height: 30)
                                    .overlay(Circle().stroke(color.hex.uppercased() == "#FFFFFF" ? Color.black : Color.clear, lineWidth: 1))
                                TextField("Nom", text: $bindableColor.name).font(.body)
                            }
                        }
                        .onDelete(perform: deleteColor)
                    }
                }
                .frame(width: 420)
                
                Divider()
                
                Form {
                    Section(header: Text("Couleurs de l'interface").font(.title3.bold())) {
                        colorPickerSection(title: "Texte général", selection: $settings.textHex)
                        colorPickerSection(title: "Bordures de la Grille", selection: $settings.gridHex)
                    }
                    Section(header: Text("Couleurs des cases").font(.title3.bold())) {
                        colorPickerSection(title: "Fond par défaut", selection: $settings.bgHex)
                        colorPickerSection(title: "Fond des Week-ends", selection: $settings.weekendBgHex)
                        colorPickerSection(title: "Fond des Jours Fériés", selection: $settings.holidayBgHex)
                    }
                }
            }
            .navigationTitle("Réglages Généraux")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .confirmationAction) {
                    Button("Fermer") { dismiss() }.font(.title3.bold())
                }
            }
        }
        .frame(width: 980, height: 700)
    }
    
    private func colorPickerSection(title: String, selection: Binding<String>) -> some View {
        VStack(alignment: .leading, spacing: 8) {
            Text(title).font(.headline)
            LazyVGrid(columns: Array(repeating: GridItem(.fixed(35), spacing: 8), count: 9), spacing: 8) {
                ForEach(palette, id: \.self) { hex in
                    Circle()
                        .fill(Color(hex: hex))
                        .frame(width: 35, height: 35)
                        .overlay(Circle().stroke(Color.blue, lineWidth: selection.wrappedValue == hex ? 3 : 0))
                        .onTapGesture { selection.wrappedValue = hex }
                }
            }
        }
        .padding(.vertical, 5)
    }
    
    private func addColor() {
        let newColor = CustomColor(name: newColorName, hex: newColorHex)
        modelContext.insert(newColor)
        newColorName = ""
    }
    
    private func deleteColor(at offsets: IndexSet) {
        for index in offsets { modelContext.delete(customColors[index]) }
    }
}

// MARK: - 5. FORMULAIRE RDV COMPACT (Style Graphique Robuste pour iPad/Mac)
struct AppointmentFormView: View {
    @Environment(\.modelContext) private var modelContext
    @Environment(\.dismiss) private var dismiss
    
    var selectedDate: Date
    var appointmentToEdit: Appointment?
    var availableColors: [CustomColor]
    
    @State private var title = ""
    @State private var timeDate = Date()
    @State private var selectedColorHex = "#FF3B30"
    @State private var location = ""
    @State private var notes = ""
    @State private var selectedIcon = "calendar"
    
    let iconOptions = [
        "calendar",
        "stethoscope",
        "breadcrumbs",
        "eyeglasses",
        "pills.fill",
        "cross.case.fill",
        "briefcase.fill",
        "house.fill",
        "scissors",
        "wrench.and.screwdriver.fill",
        "phone.fill",
        "envelope.fill",
        "cart.fill"
    ]

    var body: some View {
        NavigationStack {
            Form {
                // SECTION 1 : Titre & Horloge Graphique Intégrée (Modifiable sur iPad/Mac)
                Section(header: Text("Informations principales").font(.headline)) {
                    TextField("Titre ou description", text: $title)
                        .font(.title3)
                        .padding(.vertical, 4)
                    
                    VStack(alignment: .leading, spacing: 10) {
                        Text("Heure du RDV").font(.subheadline).foregroundColor(.secondary)
                        DatePicker("", selection: $timeDate, displayedComponents: .hourAndMinute)
                            .datePickerStyle(.graphical)
                            .labelsHidden()
                            .frame(maxWidth: .infinity)
                    }
                    .padding(.vertical, 5)
                }
                
                // SECTION 2 : Détails & Notes
                Section(header: Text("Détails géographiques & Notes").font(.headline)) {
                    HStack {
                        Image(systemName: "mappin.and.ellipse").foregroundColor(.secondary)
                        TextField("Lieu, adresse ou lien vidéo (Zoom, Meet...)", text: $location)
                    }
                    
                    VStack(alignment: .leading, spacing: 5) {
                        Text("Notes complémentaires :").font(.subheadline).foregroundColor(.secondary)
                        TextEditor(text: $notes)
                            .frame(minHeight: 80)
                            .cornerRadius(6)
                    }
                }
                
                // SECTION 3 : Choix de l'icône
                Section(header: Text("Choisir une icône illustrative").font(.headline)) {
                    ScrollView(.horizontal, showsIndicators: false) {
                        HStack(spacing: 15) {
                            ForEach(iconOptions, id: \.self) { icon in
                                Image(systemName: icon)
                                    .font(.title2)
                                    .padding(10)
                                    .frame(width: 45, height: 45)
                                    .background(selectedIcon == icon ? Color.blue.opacity(0.15) : Color.gray.opacity(0.08))
                                    .foregroundColor(selectedIcon == icon ? .blue : .primary)
                                    .clipShape(RoundedRectangle(cornerRadius: 10))
                                    .overlay(
                                        RoundedRectangle(cornerRadius: 10)
                                            .stroke(Color.blue, lineWidth: selectedIcon == icon ? 2 : 0)
                                    )
                                    .onTapGesture { selectedIcon = icon }
                            }
                        }
                        .padding(.vertical, 5)
                    }
                }
                
                // SECTION 4 : Catégorie de Couleur
                Section(header: Text("Choisir une catégorie").font(.headline)) {
                    if availableColors.isEmpty {
                        Text("Allez dans les réglages pour ajouter des couleurs").font(.body).foregroundColor(.secondary)
                    } else {
                        ForEach(availableColors) { color in
                            HStack {
                                Circle()
                                    .fill(Color(hex: color.hex))
                                    .frame(width: 30, height: 30)
                                    .overlay(Circle().stroke(color.hex.uppercased() == "#FFFFFF" ? Color.black : Color.clear, lineWidth: 1))
                                
                                Text(color.name).font(.body)
                                Spacer()
                                if selectedColorHex == color.hex {
                                    Image(systemName: "checkmark").bold().foregroundColor(.blue)
                                }
                            }
                            .contentShape(Rectangle())
                            .onTapGesture { selectedColorHex = color.hex }
                        }
                    }
                }
                
                if appointmentToEdit != nil {
                    Section {
                        Button(role: .destructive, action: deleteAppointment) {
                            HStack { Spacer(); Text("Supprimer ce rendez-vous").bold(); Spacer() }
                        }
                    }
                }
            }
            .navigationTitle(appointmentToEdit == nil ? "Nouveau RDV" : "Modifier RDV")
            .toolbar {
                ToolbarItem(placement: .cancellationAction) { Button("Annuler") { dismiss() }.font(.title3) }
                ToolbarItem(placement: .confirmationAction) {
                    Button("Enregistrer") { saveAppointment() }.font(.title3.bold()).disabled(title.isEmpty)
                }
            }
            .onAppear {
                if let rdv = appointmentToEdit {
                    title = rdv.title
                    timeDate = rdv.date
                    selectedColorHex = rdv.colorHex
                    location = rdv.location
                    notes = rdv.notes
                    selectedIcon = rdv.iconName
                } else {
                    timeDate = selectedDate
                    if let firstColor = availableColors.first { selectedColorHex = firstColor.hex }
                    location = ""
                    notes = ""
                    selectedIcon = "calendar"
                }
            }
        }
        .frame(width: 600, height: 750)
    }
    
    private func saveAppointment() {
        if let rdv = appointmentToEdit {
            rdv.title = title
            rdv.date = timeDate
            rdv.colorHex = selectedColorHex
            rdv.location = location
            rdv.notes = notes
            rdv.iconName = selectedIcon
        } else {
            let newRdv = Appointment(title: title, date: timeDate, colorHex: selectedColorHex, location: location, notes: notes, iconName: selectedIcon)
            modelContext.insert(newRdv)
        }
        dismiss()
    }
    
    private func deleteAppointment() {
        if let rdv = appointmentToEdit { modelContext.delete(rdv) }
        dismiss()
    }
}

// MARK: - EXTENSION HEX & ACCESSIBILITÉ
extension Color {
    init(hex: String) {
        let hex = hex.trimmingCharacters(in: CharacterSet.alphanumerics.inverted)
        var int: UInt64 = 0
        Scanner(string: hex).scanHexInt64(&int)
        let a, r, g, b: UInt64
        switch hex.count {
        case 3:
            (a, r, g, b) = (255, (int >> 8) * 17, (int >> 4 & 0xF) * 17, (int & 0xF) * 17)
        case 6:
            (a, r, g, b) = (255, int >> 16, int >> 8 & 0xFF, int & 0xFF)
        default:
            (a, r, g, b) = (255, 0, 0, 0)
        }
        self.init(.sRGB, red: Double(r) / 255, green: Double(g) / 255, blue: Double(b) / 255, opacity: Double(a) / 255)
    }
    
    var accessibleTextColor: Color {
        guard let cgColor = self.cgColor else { return .black }
        guard let sRGBColor = cgColor.converted(to: CGColorSpaceCreateDeviceRGB(), intent: .defaultIntent, options: nil),
              let components = sRGBColor.components, components.count >= 3 else {
            return .black
        }
        let red = components[0]
        let green = components[1]
        let blue = components[2]
        let luminance = 0.299 * red + 0.587 * green + 0.114 * blue
        return luminance < 0.45 ? .white : .black
    }
}-calendrier
