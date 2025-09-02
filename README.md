import java.io.*;
import java.nio.file.*;
import java.time.*;
import java.time.format.DateTimeFormatter;
import java.util.*;
import java.util.stream.Collectors;

public class Main {
    private static final String DATA_FILE = "events.data";
    private static final Scanner scanner = new Scanner(System.in);
    private static EventManager manager = new EventManager();
    private static User currentUser = null;

    public static void main(String[] args) {
        try {
            manager.loadFromFile(DATA_FILE);
        } catch (IOException e) {
            System.out.println("Arquivo de dados não encontrado — iniciando com lista vazia.");
        }

        mainMenuLoop();

        try {
            manager.saveToFile(DATA_FILE);
            System.out.println("Eventos salvos em '" + DATA_FILE + "'. Até mais!");
        } catch (IOException e) {
            System.out.println("Erro ao salvar eventos: " + e.getMessage());
        }
    }

    private static void mainMenuLoop() {
        boolean running = true;
        while (running) {
            System.out.println("\n=== SISTEMA DE EVENTOS - MENU PRINCIPAL ===");
            System.out.println("Usuário atual: " + (currentUser == null ? "(nenhum)" : currentUser.getDisplayName()));
            System.out.println("1) Criar/entrar usuário");
            System.out.println("2) Cadastrar evento");
            System.out.println("3) Listar eventos (todos / ordenados)");
            System.out.println("4) Visualizar eventos por status (próximos / ocorrendo / passados)");
            System.out.println("5) Confirmar participação em evento");
            System.out.println("6) Cancelar participação em evento");
            System.out.println("7) Meus eventos (confirmados)");
            System.out.println("8) Salvar agora");
            System.out.println("9) Sair");
            System.out.print("Escolha uma opção: ");
            String op = scanner.nextLine().trim();
            switch (op) {
                case "1": userMenu(); break;
                case "2": createEventMenu(); break;
                case "3": listAllEvents(); break;
                case "4": listByStatusMenu(); break;
                case "5": joinEventMenu(); break;
                case "6": leaveEventMenu(); break;
                case "7": viewMyEvents(); break;
                case "8": saveNow(); break;
                case "9": running = false; break;
                default: System.out.println("Opção inválida.");
            }
        }
    }

    private static void userMenu() {
        System.out.println("\n--- USUÁRIO ---");
        System.out.print("Digite seu nome completo: ");
        String nome = scanner.nextLine().trim();
        System.out.print("Digite seu email: ");
        String email = scanner.nextLine().trim();
        System.out.print("Digite seu telefone: ");
        String telefone = scanner.nextLine().trim();
        currentUser = new User(UUID.randomUUID().toString(), nome, email, telefone);
        System.out.println("Usuário autenticado como: " + currentUser.getDisplayName());
    }

    private static void createEventMenu() {
        if (currentUser == null) {
            System.out.println("Crie/entre com um usuário antes de cadastrar eventos (opção 1).");
            return;
        }
        System.out.println("\n--- CADASTRAR EVENTO ---");
        System.out.print("Nome do evento: ");
        String nome = scanner.nextLine().trim();
        System.out.print("Endereço: ");
        String endereco = scanner.nextLine().trim();
        System.out.println("Categorias disponíveis: " + Arrays.toString(EventCategory.values()));
        System.out.print("Escolha categoria (ex: PARTY): ");
        String catInput = scanner.nextLine().trim().toUpperCase();
        EventCategory categoria;
        try { categoria = EventCategory.valueOf(catInput); }
        catch (Exception e) { System.out.println("Categoria inválida — usando OTHER."); categoria = EventCategory.OTHER; }
        System.out.print("Data e hora (formato yyyy-MM-dd HH:mm, ex: 2025-09-01 20:30): ");
        String dtStr = scanner.nextLine().trim();
        LocalDateTime horario;
        try {
            horario = LocalDateTime.parse(dtStr.replace(' ', 'T'));
        } catch (Exception e) {
            System.out.println("Formato inválido. Evento não criado.");
            return;
        }
        System.out.print("Descrição: ");
        String descricao = scanner.nextLine().trim();

        Event ev = new Event(UUID.randomUUID().toString(), nome, endereco, categoria, horario, descricao, currentUser.getId());
        manager.addEvent(ev);
        System.out.println("Evento criado com ID: " + ev.getId());
    }

    private static void listAllEvents() {
        manager.updateEventStatuses();
        List<Event> sorted = manager.getAllEventsSortedByTime();
        if (sorted.isEmpty()) { System.out.println("Nenhum evento cadastrado."); return; }
        System.out.println("\n--- TODOS OS EVENTOS (ordenados por horário) ---");
        for (Event e : sorted) {
            System.out.println(e.displayShort());
        }
    }

    private static void listByStatusMenu() {
        manager.updateEventStatuses();
        System.out.println("\n1) Próximos (futuros)");
        System.out.println("2) Ocorrendo agora");
        System.out.println("3) Já ocorreram (passados)");
        System.out.print("Escolha: ");
        String o = scanner.nextLine().trim();
        switch (o) {
            case "1": listEventsFiltered(e -> e.getStatus() == EventStatus.UPCOMING); break;
            case "2": listEventsFiltered(e -> e.getStatus() == EventStatus.ONGOING); break;
            case "3": listEventsFiltered(e -> e.getStatus() == EventStatus.PAST); break;
            default: System.out.println("Opção inválida.");
        }
    }

    private static void listEventsFiltered(java.util.function.Predicate<Event> pred) {
        List<Event> lst = manager.getAllEventsSortedByTime().stream().filter(pred).collect(Collectors.toList());
        if (lst.isEmpty()) { System.out.println("Nenhum evento nessa categoria."); return; }
        for (Event e : lst) System.out.println(e.displayFull());
    }

    private static void joinEventMenu() {
        if (currentUser == null) { System.out.println("Faça login (opção 1) antes de confirmar presença."); return; }
        System.out.print("Digite o ID do evento que deseja participar: ");
        String id = scanner.nextLine().trim();
        try {
            manager.confirmParticipation(id, currentUser.getId());
            System.out.println("Participação confirmada no evento " + id + ".");
        } catch (IllegalArgumentException e) { System.out.println("Erro: " + e.getMessage()); }
    }

    private static void leaveEventMenu() {
        if (currentUser == null) { System.out.println("Faça login (opção 1) antes de cancelar presença."); return; }
        System.out.print("Digite o ID do evento que deseja cancelar participação: ");
        String id = scanner.nextLine().trim();
        try {
            manager.cancelParticipation(id, currentUser.getId());
            System.out.println("Participação cancelada no evento " + id + ".");
        } catch (IllegalArgumentException e) { System.out.println("Erro: " + e.getMessage()); }
    }

    private static void viewMyEvents() {
        if (currentUser == null) { System.out.println("Faça login (opção 1) antes de ver seus eventos."); return; }
        List<Event> lst = manager.getEventsWhereUserParticipates(currentUser.getId());
        if (lst.isEmpty()) { System.out.println("Você não confirmou presença em nenhum evento."); return; }
        System.out.println("\n--- MEUS EVENTOS CONFIRMADOS ---");
        for (Event e : lst) System.out.println(e.displayFull());
    }

    private static void saveNow() {
        try {
            manager.saveToFile(DATA_FILE);
            System.out.println("Dados salvos com sucesso.");
        } catch (IOException e) {
            System.out.println("Erro ao salvar: " + e.getMessage());
        }
    }
}

class User {
    private final String id;
    private final String nome;
    private final String email;
    private final String telefone;

    public User(String id, String nome, String email, String telefone) {
        this.id = id; this.nome = nome; this.email = email; this.telefone = telefone;
    }

    public String getId() { return id; }
    public String getNome() { return nome; }
    public String getEmail() { return email; }
    public String getTelefone() { return telefone; }
    public String getDisplayName() { return nome + " <" + email + ">"; }
}

enum EventCategory { PARTY, SPORTS, SHOW, CONFERENCE, MEETING, OTHER }

enum EventStatus { UPCOMING, ONGOING, PAST }

class Event {
    private final String id;
    private final String name;
    private final String address;
    private final EventCategory category;
    private final LocalDateTime datetime;
    private final String description;
    private final String creatorUserId;
    private final Set<String> participants = new HashSet<>();

    public Event(String id, String name, String address, EventCategory category, LocalDateTime datetime, String description, String creatorUserId) {
        this.id = id; this.name = name; this.address = address; this.category = category; this.datetime = datetime; this.description = description; this.creatorUserId = creatorUserId;
    }

    public String getId() { return id; }
    public String getName() { return name; }
    public String getAddress() { return address; }
    public EventCategory getCategory() { return category; }
    public LocalDateTime getDatetime() { return datetime; }
    public String getDescription() { return description; }
    public Set<String> getParticipants() { return participants; }
    public String getCreatorUserId() { return creatorUserId; }

    public EventStatus getStatus() {
        LocalDateTime now = LocalDateTime.now();
        LocalDateTime start = datetime;
        LocalDateTime end = datetime.plusHours(2);
        if (now.isBefore(start)) return EventStatus.UPCOMING;
        if (!now.isBefore(start) && now.isBefore(end)) return EventStatus.ONGOING;
        return EventStatus.PAST;
    }

    public String serialize() {
        String participantsStr = String.join(",", participants);
        return String.join("|",
            id,
            escape(name),
            escape(address),
            category.name(),
            datetime.toString(),
            escape(description),
            participantsStr
        );
    }

    private String escape(String s) {
        return s.replace("|", " ").replace("\n", " ");
    }

    public static Event deserialize(String line) throws IllegalArgumentException {
        String[] parts = line.split("\\|", -1);
        if (parts.length < 7) throw new IllegalArgumentException("Linha de dados inválida");
        String id = parts[0];
        String name = parts[1];
        String address = parts[2];
        EventCategory category = EventCategory.valueOf(parts[3]);
        LocalDateTime dt = LocalDateTime.parse(parts[4]);
        String desc = parts[5];
        Event ev = new Event(id, name, address, category, dt, desc, "(imported)");
        if (!parts[6].isEmpty()) {
            String[] p = parts[6].split(",");
            for (String pid : p) if (!pid.trim().isEmpty()) ev.participants.add(pid.trim());
        }
        return ev;
    }

    public String displayShort() {
        DateTimeFormatter f = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm");
        return String.format("[%s] %s — %s (%s) — %s — participantes: %d",
            id, name, address, category.name(), datetime.format(f), participants.size());
    }

    public String displayFull() {
        DateTimeFormatter f = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm");
        return String.format("ID: %s\nNome: %s\nEndereço: %s\nCategoria: %s\nHorário: %s\nDescrição: %s\nParticipantes: %s\nStatus: %s\n",
            id, name, address, category.name(), datetime.format(f), description, participants.isEmpty() ? "(nenhum)" : String.join(", ", participants), getStatus().name());
    }
}

class EventManager {
    private final Map<String, Event> events = new HashMap<>();

    public void addEvent(Event e) {
        events.put(e.getId(), e);
    }

    public List<Event> getAllEventsSortedByTime() {
        return events.values().stream()
            .sorted(Comparator.comparing(Event::getDatetime))
            .collect(Collectors.toList());
    }

    public void confirmParticipation(String eventId, String userId) {
        Event e = events.get(eventId);
        if (e == null) throw new IllegalArgumentException("Evento não encontrado.");
        if (e.getStatus() == EventStatus.PAST) throw new IllegalArgumentException("Não é possível confirmar presença em evento que já ocorreu.");
        if (e.getParticipants().contains(userId)) throw new IllegalArgumentException("Você já confirmou presença neste evento.");
        e.getParticipants().add(userId);
    }

    public void cancelParticipation(String eventId, String userId) {
        Event e = events.get(eventId);
        if (e == null) throw new IllegalArgumentException("Evento não encontrado.");
        if (!e.getParticipants().remove(userId)) throw new IllegalArgumentException("Você não estava participando deste evento.");
    }

    public List<Event> getEventsWhereUserParticipates(String userId) {
        return events.values().stream().filter(ev -> ev.getParticipants().contains(userId)).sorted(Comparator.comparing(Event::getDatetime)).collect(Collectors.toList());
    }

    public void saveToFile(String filename) throws IOException {
        List<String> lines = getAllEventsSortedByTime().stream().map(Event::serialize).collect(Collectors.toList());
        Path p = Paths.get(filename);
        Files.write(p, lines, StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING);
    }

    public void loadFromFile(String filename) throws IOException {
        Path p = Paths.get(filename);
        if (!Files.exists(p)) throw new IOException("Arquivo não existe");
        List<String> lines = Files.readAllLines(p);
        for (String l : lines) {
            if (l.trim().isEmpty()) continue;
            try {
                Event ev = Event.deserialize(l);
                events.put(ev.getId(), ev);
            } catch (Exception e) {
                System.out.println("Ignorando linha inválida: " + l);
            }
        }
    }

    public void updateEventStatuses() {
    }
}
