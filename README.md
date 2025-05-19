# Model

import SwiftUI
import CloudKit
import UserNotifications

struct Transaction: Identifiable, Codable {
    let id: UUID
    let senderId: Int
    let receiverId: Int
    let amount: Double
    let timestamp: Date
}

struct TransactionRecord: Identifiable, Codable {
    let id: UUID
    let senderName: String
    let receiverName: String
    let amount: Double
    let timestamp: Date
}

struct Account: Identifiable, Codable {
    let id: Int
    var name: String
    var balance: Double
    var transactions: [Transaction]
    var password: String
}

class AccountManager: ObservableObject {
    @Published var accounts = [Account]()
    @Published var transactionRecords = [TransactionRecord]()
    private var privateDB = CKContainer.default().privateCloudDatabase
    
    init() {
        fetchAccountsFromCloud()
        fetchTransactionRecordsFromCloud()
    }
    
    func saveAccountToCloud(_ account: Account) {
        let recordID = CKRecord.ID(recordName: "\(account.id)")
        let record = CKRecord(recordType: "Account", recordID: recordID)
        record["name"] = account.name as CKRecordValue
        record["balance"] = account.balance as CKRecordValue
        privateDB.save(record) { (savedRecord, error) in
            if let error = error {
                print("Erro ao salvar account no CloudKit: \(error.localizedDescription)")
            } else {
                print("Account \(account.id) salva no CloudKit com sucesso!")
            }
        }
    }
    
    func fetchAccountsFromCloud() {
        let query = CKQuery(recordType: "Account", predicate: NSPredicate(value: true))
        privateDB.perform(query, inZoneWith: nil) { records, error in
            if let records = records {
                let fetchedAccounts = records.compactMap { record -> Account? in
                    guard let name = record["name"] as? String,
                          let balance = record["balance"] as? Double,
                          let id = Int(record.recordID.recordName)
                    else { return nil }
                    return Account(id: id, name: name, balance: balance, transactions: [], password: "")
                }
                DispatchQueue.main.async {
                    if fetchedAccounts.isEmpty {
                        self.accounts = [
                            Account(id: 1234, name: "Mariana Silva", balance: 0, transactions: [], password: "senha1234"),
                            Account(id: 2345, name: "Samuel Campos", balance: 0.0, transactions: [], password: "senha2345"),
                            Account(id: 40800, name: "Maria Eugenia", balance: 0.0, transactions: [], password: "senha40800"),
                            Account(id: 3208, name: "BeBelle", balance: 0.0, transactions: [], password: "senha2345"),
                            Account(id: 3847, name: "Nice Tigrinho", balance: 0.0, transactions: [], password: "senha3847"),
                            Account(id: 0, name: "iPay", balance: .infinity, transactions: [], password: "senhabank")
                        ]
                        self.accounts.forEach { self.saveAccountToCloud($0) }
                    } else {
                        self.accounts = fetchedAccounts
                    }
                }
            } else if let error = error {
                print("Erro ao buscar accounts do CloudKit: \(error.localizedDescription)")
            }
        }
    }
    
    func saveTransactionRecordToCloud(_ record: TransactionRecord) {
        let recordID = CKRecord.ID(recordName: record.id.uuidString)
        let ckRecord = CKRecord(recordType: "TransactionRecord", recordID: recordID)
        ckRecord["senderName"] = record.senderName as CKRecordValue
        ckRecord["receiverName"] = record.receiverName as CKRecordValue
        ckRecord["amount"] = record.amount as CKRecordValue
        ckRecord["timestamp"] = record.timestamp as CKRecordValue
        privateDB.save(ckRecord) { (_, error) in
            if let error = error {
                print("Erro ao salvar registro de transação no CloudKit: \(error.localizedDescription)")
            } else {
                print("Registro de transação salvo com sucesso no CloudKit!")
            }
        }
    }
    
    func fetchTransactionRecordsFromCloud() {
        let query = CKQuery(recordType: "TransactionRecord", predicate: NSPredicate(value: true))
        privateDB.perform(query, inZoneWith: nil) { records, error in
            if let records = records {
                let fetchedRecords = records.compactMap { record -> TransactionRecord? in
                    guard let senderName = record["senderName"] as? String,
                          let receiverName = record["receiverName"] as? String,
                          let amount = record["amount"] as? Double,
                          let timestamp = record["timestamp"] as? Date,
                          let id = UUID(uuidString: record.recordID.recordName)
                    else { return nil }
                    return TransactionRecord(id: id, senderName: senderName, receiverName: receiverName, amount: amount, timestamp: timestamp)
                }
                DispatchQueue.main.async {
                    self.transactionRecords = fetchedRecords
                }
            } else if let error = error {
                print("Erro ao buscar registros de transação do CloudKit: \(error.localizedDescription)")
            }
        }
    }
    
    func saveTransactionRecord(senderName: String, receiverName: String, amount: Double) {
        let transactionRecord = TransactionRecord(id: UUID(), senderName: senderName, receiverName: receiverName, amount: amount, timestamp: Date())
        transactionRecords.append(transactionRecord)
        saveTransactionRecordToCloud(transactionRecord)
    }
    
    func authenticate(accountId: Int, password: String) -> Bool {
        if let account = accounts.first(where: { $0.id == accountId && $0.password == password }) {
            return true
        }
        return false
    }
    
    func logout() {
    }
    
    func sendTransaction(receiverId: Int, amount: Double) {
        guard let senderAccount = accounts.first(where: { $0.id != receiverId }) else { return }
        guard let receiverAccount = accounts.first(where: { $0.id == receiverId }) else { return }
        let senderTransaction = Transaction(id: UUID(), senderId: senderAccount.id, receiverId: receiverAccount.id, amount: -amount, timestamp: Date())
        let receiverTransaction = Transaction(id: UUID(), senderId: senderAccount.id, receiverId: receiverAccount.id, amount: amount, timestamp: Date())
        if let senderIndex = accounts.firstIndex(where: { $0.id == senderAccount.id }) {
            accounts[senderIndex].transactions.append(senderTransaction)
        }
        if let receiverIndex = accounts.firstIndex(where: { $0.id == receiverAccount.id }) {
            accounts[receiverIndex].transactions.append(receiverTransaction)
        }
        saveTransactionRecord(senderName: senderAccount.name, receiverName: receiverAccount.name, amount: amount)
        sendPushNotification(to: receiverId, title: "Nova Transação", body: "Você recebeu $\(amount) de \(senderAccount.name)")
    }
    
    func sendPushNotification(to receiverId: Int, title: String, body: String) {
        let content = UNMutableNotificationContent()
        content.title = title
        content.body = body
        let request = UNNotificationRequest(identifier: UUID().uuidString, content: content, trigger: nil)
        UNUserNotificationCenter.current().add(request)
    }
    
    func sendAdminPushNotification(to receiverId: Int, title: String, body: String) {
        sendPushNotification(to: receiverId, title: title, body: body)
    }
    
    func syncBalance() {
        fetchAccountsFromCloud()
        print("Sim, o saldo pode ser atualizado em outro dispositivo desde que você implemente a sincronização adequada usando o SwiftCloud. Ou seja, quando uma operação que altera o saldo é enviada e salva na nuvem, outro dispositivo que fizer o fetch desses registros receberá o saldo atualizado. Sem um mecanismo de notificação ou atualização automática, o dispositivo precisará executar uma ação—como, por exemplo, chamar um método de busca ou sincronização—para carregar as últimas informações")
    }
}

struct ContentView: View {
    @StateObject private var accountManager = AccountManager()
    @State private var accountId: Int?
    @State private var password: String = ""
    
    var body: some View {
        NavigationStack {
            if accountManager.accounts.contains(where: { $0.id == accountId ?? -1 }) {
                MainView(accountManager: accountManager)
            } else {
                LoginView(accountId: $accountId, password: $password, accountManager: accountManager)
            }
        }
        .onAppear {
            UNUserNotificationCenter.current().requestAuthorization(options: [.alert, .badge, .sound]) { _, _ in }
        }
    }
}

struct LoginView: View {
    @Binding var accountId: Int?
    @Binding var password: String
    @ObservedObject var accountManager: AccountManager
    
    var body: some View {
        ScrollView {
            Text("Seu ID no iPay:")
                .font(.custom("NoteWorthy", size: 23.83))
                .foregroundColor(.cyan)
            TextField("Seu ID", text: accountIdText)
                .textFieldStyle(RoundedBorderTextFieldStyle())
                .keyboardType(.numberPad)
                .padding()
                .font(.custom("NoteWorthy", size: 23.83))
                .foregroundColor(.cyan)
            Text("Sua Senha:")
                .font(.custom("NoteWorthy", size: 23.83))
                .foregroundColor(.cyan)
            SecureField("Senha", text: $password)
                .textFieldStyle(RoundedBorderTextFieldStyle())
                .padding()
                .font(.custom("NoteWorthy", size: 23.83))
                .foregroundColor(.cyan)
            Button(action: authenticate) {
                Text("Login")
                    .padding()
                    .foregroundColor(.white)
                    .background(Color.cyan)
                    .cornerRadius(10)
                    .font(.custom("NoteWorthy", size: 23.83))
            }
            .padding()
        }
        .padding()
        .edgesIgnoringSafeArea(.all)
    }
    
    private var accountIdText: Binding<String> {
        Binding {
            accountId?.description ?? ""
        } set: { newValue in
            accountId = Int(newValue)
        }
    }
    
    private func authenticate() {
        if let accountId = accountId, accountManager.authenticate(accountId: accountId, password: password) {
            return
        }
    }
}

struct MainView: View {
    @ObservedObject var accountManager: AccountManager
    
    var body: some View {
        VStack {
            Text("$\(accountManager.accounts.first(where: { $0.id == (accountManager.accounts.first?.id ?? 0) })?.balance ?? 0, specifier: "%.2f")")
                .font(.custom("NoteWorthy", size: 40))
                .bold()
                .foregroundColor(.cyan)
                .padding()
            Divider()
            TabView {
                TransactionView(accountManager: accountManager)
                    .tabItem {
                        Image(systemName: "creditcard")
                    }
                ExtratoView(transactionRecords: accountManager.transactionRecords, accountManager: accountManager)
                    .tabItem {
                        Image(systemName: "list.bullet")
                    }
                SyncView(accountManager: accountManager)
                    .tabItem {
                        Image(systemName: "arrow.clockwise")
                    }
                LogoutView(accountManager: accountManager)
                    .tabItem {
                        Image(systemName: "arrow.backward")
                    }
            }
        }
    }
}

struct TransactionView: View {
    @ObservedObject var accountManager: AccountManager
    @State private var amountString = ""
    @State private var receiverAccountId: Int?
    
    var body: some View {
        VStack {
            Text("ID da conta destinatária:")
                .font(.custom("Noteworthy", size: 16.8))
                .foregroundColor(.cyan)
            TextField("ID", text: accountIdText)
                .textFieldStyle(RoundedBorderTextFieldStyle())
                .keyboardType(.numberPad)
                .padding()
                .font(.custom("NoteWorthy", size: 33.2))
                .foregroundColor(.cyan)
            Text("Valor a ser enviado:")
                .font(.custom("NoteWorthy", size: 16.8))
                .foregroundColor(.cyan)
            HStack {
                Text("$")
                    .font(.custom("NoteWorthy", size: 35.7))
                    .foregroundColor(.cyan)
                    .bold()
                TextField("Valor", text: $amountString)
                    .textFieldStyle(RoundedBorderTextFieldStyle())
                    .keyboardType(.decimalPad)
                    .padding()
                    .font(.custom("NoteWorthy", size: 35.7))
                    .foregroundColor(.cyan)
            }
            .padding()
            Button(action: sendTransaction) {
                Text("Enviar")
                    .padding()
                    .foregroundColor(.white)
                    .background(Color.cyan)
                    .cornerRadius(10)
                    .font(.custom("NoteWorthy", size: 16.8))
            }
            .padding()
            Spacer()
        }
        .padding()
        .edgesIgnoringSafeArea(.all)
    }
    
    private var accountIdText: Binding<String> {
        Binding {
            receiverAccountId?.description ?? ""
        } set: { newValue in
            receiverAccountId = Int(newValue)
        }
    }
    
    private func sendTransaction() {
        guard let amount = Double(amountString), let receiverId = receiverAccountId else { return }
        accountManager.sendTransaction(receiverId: receiverId, amount: amount)
        amountString = ""
        receiverAccountId = nil
    }
}

struct ExtratoView: View {
    var transactionRecords: [TransactionRecord]
    var accountManager: AccountManager
    
    var body: some View {
        VStack {
            List(transactionRecords) { record in
                if record.senderName == accountManager.accounts.first(where: { $0.id == (accountManager.accounts.first?.id ?? 0) })?.name || record.receiverName == accountManager.accounts.first(where: { $0.id == (accountManager.accounts.first?.id ?? 0) })?.name || (accountManager.accounts.first?.id ?? 0) == 91011 {
                    VStack(alignment: .leading) {
                        Text("De: \(record.senderName)")
                        Text("Para: \(record.receiverName)")
                        Text("Valor: \(record.amount, specifier: "%.2f")")
                    }
                }
            }
        }
    }
}

struct SyncView: View {
    @ObservedObject var accountManager: AccountManager
    var body: some View {
        VStack {
            Button("Sincronizar Saldo") {
                accountManager.syncBalance()
            }
            .padding()
            .background(Color.green)
            .foregroundColor(.white)
            .cornerRadius(8)
        }
    }
}

struct AdminPushNotificationView: View {
    @ObservedObject var accountManager: AccountManager
    @State private var receiverAccountId: Int?
    @State private var title: String = ""
    @State private var notificationBody: String = ""
    
    var body: some View {
        VStack {
            Text("ID da conta destinatária:")
                .font(.custom("Impact", size: 16.8))
                .foregroundColor(.cyan)
            TextField("ID", text: accountIdText)
                .textFieldStyle(RoundedBorderTextFieldStyle())
                .keyboardType(.numberPad)
                .padding()
                .font(.custom("NoteWorthy", size: 23.83))
                .foregroundColor(.cyan)
            Text("Título da notificação:")
                .font(.custom("Impact", size: 16.8))
                .foregroundColor(.cyan)
            TextField("Título", text: $title)
                .textFieldStyle(RoundedBorderTextFieldStyle())
                .padding()
                .font(.custom("NoteWorthy", size: 23.83))
                .foregroundColor(.cyan)
            Text("Corpo da notificação:")
                .font(.custom("Impact", size: 16.8))
                .foregroundColor(.cyan)
            TextField("Corpo", text: $notificationBody)
                .textFieldStyle(RoundedBorderTextFieldStyle())
                .padding()
                .font(.custom("NoteWorthy", size: 23.83))
                .foregroundColor(.cyan)
            Button(action: sendNotification) {
                Text("Enviar Notificação")
                    .padding()
                    .foregroundColor(.white)
                    .background(Color.cyan)
                    .cornerRadius(10)
                    .font(.custom("NoteWorthy", size: 16.8))
            }
            .padding()
            Spacer()
        }
        .padding()
        .edgesIgnoringSafeArea(.all)
    }
    
    private var accountIdText: Binding<String> {
        Binding {
            receiverAccountId?.description ?? ""
        } set: { newValue in
            receiverAccountId = Int(newValue)
        }
    }
    
    private func sendNotification() {
        guard let receiverId = receiverAccountId, !title.isEmpty, !notificationBody.isEmpty else { return }
        accountManager.sendAdminPushNotification(to: receiverId, title: title, body: notificationBody)
        title = ""
        notificationBody = ""
        receiverAccountId = nil
    }
}

struct LogoutView: View {
    @ObservedObject var accountManager: AccountManager
    
    var body: some View {
        Button(action: logout) {
            Text("Logout")
                .padding()
                .foregroundColor(.white)
                .background(Color.cyan)
                .cornerRadius(10)
                .font(.custom("NoteWorthy", size: 23.83))
        }
        .padding()
    }
    
    private func logout() {
        accountManager.logout()
    }
}




import SwiftUI
import CloudKit

@main
struct MyApp: App {
    init() {
        let container = CKContainer.default()
        container.accountStatus { status, error in
            switch status {
            case .available:
                print("CloudKit disponível")
            default:
                print("CloudKit indisponível: \(status)")
            }
            if let error = error {
                print("Erro no CloudKit: \(error.localizedDescription)")
            }
        }
    }
    
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}
