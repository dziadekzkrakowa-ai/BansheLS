package pl.banshels;

import org.bukkit.OfflinePlayer;
import org.bukkit.Particle;
import org.bukkit.World;
import org.bukkit.event.player.PlayerEditBookEvent;
import org.bukkit.inventory.meta.BookMeta;
import org.bukkit.scheduler.BukkitRunnable;
import org.bukkit.util.Vector;
import java.util.concurrent.ThreadLocalRandom;
import io.papermc.paper.datacomponent.DataComponentTypes;
import io.papermc.paper.datacomponent.item.DeathProtection;
import java.io.File;
import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.nio.file.AtomicMoveNotSupportedException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.StandardCopyOption;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.logging.Level;
import net.kyori.adventure.text.Component;
import net.kyori.adventure.text.format.TextDecoration;
import net.kyori.adventure.text.serializer.legacy.LegacyComponentSerializer;
import org.bukkit.Bukkit;
import org.bukkit.EntityEffect;
import org.bukkit.Location;
import org.bukkit.Material;
import org.bukkit.NamespacedKey;
import org.bukkit.Sound;
import org.bukkit.attribute.Attribute;
import org.bukkit.attribute.AttributeInstance;
import org.bukkit.command.Command;
import org.bukkit.command.CommandExecutor;
import org.bukkit.command.CommandSender;
import org.bukkit.command.PluginCommand;
import org.bukkit.command.TabCompleter;
import org.bukkit.configuration.ConfigurationSection;
import org.bukkit.configuration.InvalidConfigurationException;
import org.bukkit.configuration.file.YamlConfiguration;
import org.bukkit.entity.Item;
import org.bukkit.entity.Player;
import org.bukkit.event.EventHandler;
import org.bukkit.event.EventPriority;
import org.bukkit.event.Listener;
import org.bukkit.event.block.Action;
import org.bukkit.event.block.BlockDispenseEvent;
import org.bukkit.event.entity.PlayerDeathEvent;
import org.bukkit.event.inventory.PrepareItemCraftEvent;
import org.bukkit.event.player.PlayerInteractAtEntityEvent;
import org.bukkit.event.player.PlayerInteractEntityEvent;
import org.bukkit.event.player.PlayerInteractEvent;
import org.bukkit.event.player.PlayerJoinEvent;
import org.bukkit.event.player.PlayerRespawnEvent;
import org.bukkit.inventory.EquipmentSlot;
import org.bukkit.inventory.ItemStack;
import org.bukkit.inventory.meta.ItemMeta;
import org.bukkit.inventory.meta.components.CustomModelDataComponent;
import org.bukkit.persistence.PersistentDataContainer;
import org.bukkit.persistence.PersistentDataType;
import org.bukkit.plugin.java.JavaPlugin;

public final class BansheLS extends JavaPlugin {

    private HeartManager manager;

    @Override
    public void onEnable() {
        manager = new HeartManager(this);
        if (!manager.load()) {
            getServer().getPluginManager().disablePlugin(this);
            return;
        }

        HeartItem items = new HeartItem(this);
        boolean invulnerable = true;
        getServer().getPluginManager().registerEvents(new HeartListener(this, manager, items, invulnerable), this);

        PluginCommand command = getCommand("give-serce");
        if (command != null) {
            GiveHeartCommand executor = new GiveHeartCommand(items);
            command.setExecutor(executor);
            command.setTabCompleter(executor);
        }

        PluginCommand withdraw = getCommand("wyplacserce");
        if (withdraw != null) {
            withdraw.setExecutor(new WithdrawHeartCommand(manager, items));
        }

        ReviveBook reviveBook = new ReviveBook(this);
        getServer().getPluginManager().registerEvents(new ReviveListener(this, manager, reviveBook), this);
        PluginCommand giveBook = getCommand("give-ksiegaodrodzenia");
        if (giveBook != null) {
            giveBook.setExecutor(new GiveReviveBookCommand(reviveBook));
        }

        // Obsluga /reload - gracze, ktorzy sa juz online.
        for (Player player : Bukkit.getOnlinePlayers()) {
            manager.ensure(player.getUniqueId());
            manager.applyMaxHealth(player);
        }
        getLogger().info("BansheLS wlaczony.");
    }

    @Override
    public void onDisable() {
        if (manager != null && manager.isLoaded()) {
            manager.save();
        }
    }
}

/** Pomocnicze metody do kolorowania tekstu kodami &. */
final class Text {

    private static final LegacyComponentSerializer AMPERSAND = LegacyComponentSerializer.legacyAmpersand();
    private static final LegacyComponentSerializer SECTION = LegacyComponentSerializer.legacySection();

    private Text() {
    }

    /** Zwykly kolorowy komponent (do wiadomosci na czacie). */
    public static Component color(String text) {
        return AMPERSAND.deserialize(text);
    }

    /** Komponent do nazwy/lore przedmiotu - bez domyslnej kursywy. */
    public static Component item(String text) {
        return AMPERSAND.deserialize(text).decoration(TextDecoration.ITALIC, false);
    }

    /** Tekst z kodami § (np. do powodu bana). */
    public static String section(String text) {
        return SECTION.serialize(AMPERSAND.deserialize(text));
    }
}

/** Tworzenie i rozpoznawanie przedmiotu serca. */
final class HeartItem {

    private final NamespacedKey key;

    public HeartItem(BansheLS plugin) {
        this.key = new NamespacedKey(plugin, "heart");
    }

    /** Tworzy przedmiot serca. Ten sam przedmiot wypada po smierci i jest dawany komenda. */
    public ItemStack create(int amount) {
        ItemStack item = new ItemStack(Material.RED_DYE, amount);
        ItemMeta meta = item.getItemMeta();

        meta.displayName(Text.item("&4&lSerce"));
        meta.lore(List.of(Text.item("&7Dodatkowe Serce")));

        CustomModelDataComponent cmd = meta.getCustomModelDataComponent();
        cmd.setFloats(List.of(1.0f));
        meta.setCustomModelDataComponent(cmd);

        meta.getPersistentDataContainer().set(key, PersistentDataType.BYTE, (byte) 1);
        item.setItemMeta(meta);
        return item;
    }

    /** Prawdziwe serce = RED_DYE + znacznik w PersistentDataContainer. Nazwa i CMD sie nie licza. */
    public boolean isHeart(ItemStack item) {
        if (item == null || item.getType() != Material.RED_DYE || !item.hasItemMeta()) {
            return false;
        }
        PersistentDataContainer pdc = item.getItemMeta().getPersistentDataContainer();
        Byte marker = pdc.get(key, PersistentDataType.BYTE);
        return marker != null && marker == (byte) 1;
    }
}

/** Przechowuje liczbe serc graczy i zapisuje ja na stale w data.yml. */
final class HeartManager {

    public static final int DEFAULT_HEARTS = 10;
    public static final int MAX_HEARTS = 20;
    public static final int MIN_WITHDRAW_HEARTS = 10;
    public static final int REVIVE_HEARTS = 10;

    private final BansheLS plugin;
    private final File file;
    private final Map<UUID, Integer> hearts = new HashMap<>();
    private boolean loaded = false;

    public HeartManager(BansheLS plugin) {
        this.plugin = plugin;
        this.file = new File(plugin.getDataFolder(), "data.yml");
    }

    public boolean isLoaded() {
        return loaded;
    }

    /** Wczytuje dane. Zwraca false, gdy plik jest uszkodzony (zeby go nie nadpisac). */
    public synchronized boolean load() {
        plugin.getDataFolder().mkdirs();
        if (file.exists()) {
            YamlConfiguration yml = new YamlConfiguration();
            try {
                yml.load(file);
            } catch (IOException | InvalidConfigurationException e) {
                plugin.getLogger().log(Level.SEVERE,
                        "Nie mozna wczytac data.yml - plugin zostaje wylaczony, aby nie utracic danych.", e);
                return false;
            }
            ConfigurationSection section = yml.getConfigurationSection("players");
            if (section != null) {
                for (String key : section.getKeys(false)) {
                    try {
                        UUID uuid = UUID.fromString(key);
                        int value = Math.max(0, Math.min(MAX_HEARTS, section.getInt(key)));
                        hearts.put(uuid, value);
                    } catch (IllegalArgumentException ex) {
                        plugin.getLogger().warning("Pominieto niepoprawny wpis w data.yml: " + key);
                    }
                }
            }
        }
        loaded = true;
        return true;
    }

    public synchronized boolean has(UUID uuid) {
        return hearts.containsKey(uuid);
    }

    public synchronized int getHearts(UUID uuid) {
        return hearts.getOrDefault(uuid, DEFAULT_HEARTS);
    }

    /** Ustawia liczbe serc (0-20) i od razu zapisuje do pliku. */
    public synchronized void setHearts(UUID uuid, int value) {
        hearts.put(uuid, Math.max(0, Math.min(MAX_HEARTS, value)));
        save();
    }

    /** Tworzy wpis dla nowego gracza (10 serc). */
    public synchronized void ensure(UUID uuid) {
        if (!hearts.containsKey(uuid)) {
            setHearts(uuid, DEFAULT_HEARTS);
        }
    }

    /** Zapis atomowy: najpierw plik tymczasowy, potem podmiana. */
    public synchronized void save() {
        if (!loaded) {
            return;
        }
        YamlConfiguration yml = new YamlConfiguration();
        for (Map.Entry<UUID, Integer> entry : hearts.entrySet()) {
            yml.set("players." + entry.getKey(), entry.getValue());
        }
        try {
            plugin.getDataFolder().mkdirs();
            Path target = file.toPath();
            Path tmp = new File(plugin.getDataFolder(), "data.yml.tmp").toPath();
            Files.writeString(tmp, yml.saveToString(), StandardCharsets.UTF_8);
            try {
                Files.move(tmp, target, StandardCopyOption.REPLACE_EXISTING, StandardCopyOption.ATOMIC_MOVE);
            } catch (AtomicMoveNotSupportedException e) {
                Files.move(tmp, target, StandardCopyOption.REPLACE_EXISTING);
            }
        } catch (IOException e) {
            plugin.getLogger().log(Level.SEVERE, "Nie udalo sie zapisac data.yml!", e);
        }
    }

    /** Ustawia maksymalne zdrowie gracza na (serca * 2). Przy 0 serc nic nie robi. */
    public void applyMaxHealth(Player player) {
        int count = getHearts(player.getUniqueId());
        if (count <= 0) {
            return;
        }
        AttributeInstance attribute = player.getAttribute(Attribute.MAX_HEALTH);
        if (attribute == null) {
            return;
        }
        double max = count * 2.0;
        attribute.setBaseValue(max);
        if (player.getHealth() > max) {
            player.setHealth(max);
        }
    }
}

final class HeartListener implements Listener {

    private static final String BAN_REASON = "&cNie masz juz serc, zostales wyzerowany";

    private final BansheLS plugin;
    private final HeartManager manager;
    private final HeartItem items;
    private final boolean invulnerableDrop;

    public HeartListener(BansheLS plugin, HeartManager manager, HeartItem items, boolean invulnerableDrop) {
        this.plugin = plugin;
        this.manager = manager;
        this.items = items;
        this.invulnerableDrop = invulnerableDrop;
    }

    // ------------------------------------------------------------ wejscie / respawn

    @EventHandler
    public void onJoin(PlayerJoinEvent event) {
        Player player = event.getPlayer();
        UUID uuid = player.getUniqueId();
        if (!manager.has(uuid)) {
            manager.ensure(uuid);
        } else if (manager.getHearts(uuid) <= 0) {
            // Gracz mial 0 serc, ale zostal odbanowany przez admina - zaczyna od nowa.
            manager.setHearts(uuid, HeartManager.DEFAULT_HEARTS);
        }
        manager.applyMaxHealth(player);
    }

    @EventHandler
    public void onRespawn(PlayerRespawnEvent event) {
        Player player = event.getPlayer();
        Bukkit.getScheduler().runTask(plugin, () -> {
            if (!player.isOnline()) {
                return;
            }
            manager.applyMaxHealth(player);
            AttributeInstance attribute = player.getAttribute(Attribute.MAX_HEALTH);
            if (attribute != null && !player.isDead()) {
                player.setHealth(attribute.getValue());
            }
        });
    }

    // ------------------------------------------------------------ smierc

    @EventHandler(priority = EventPriority.HIGHEST, ignoreCancelled = true)
    public void onDeath(PlayerDeathEvent event) {
        Player player = event.getEntity();
        UUID uuid = player.getUniqueId();

        int remaining = Math.max(0, manager.getHearts(uuid) - 1);
        manager.setHearts(uuid, remaining);
        manager.applyMaxHealth(player);

        // Serce wypada zawsze, niezaleznie od przyczyny smierci.
        Location location = player.getLocation();
        Item dropped = location.getWorld().dropItemNaturally(location, items.create(1));
        if (invulnerableDrop) {
            dropped.setInvulnerable(true);
        }

        if (remaining <= 0) {
            final String name = player.getName();
            Bukkit.getScheduler().runTaskLater(plugin, () -> banPlayer(name, uuid), 1L);
        } else {
            player.sendMessage(Text.color("&cStraciles serce! Pozostalo ci: &4" + remaining + "&c."));
        }
    }

    private void banPlayer(String name, UUID uuid) {
        String reason = Text.section(BAN_REASON);
        Bukkit.dispatchCommand(Bukkit.getConsoleSender(), "minecraft:ban " + name + " " + reason);
        Player online = Bukkit.getPlayer(uuid);
        if (online != null && online.isOnline()) {
            online.kick(Text.color(BAN_REASON));
        }
    }

    // ------------------------------------------------------------ uzywanie serca

    @EventHandler(priority = EventPriority.HIGH)
    public void onInteract(PlayerInteractEvent event) {
        Action action = event.getAction();
        if (action != Action.RIGHT_CLICK_AIR && action != Action.RIGHT_CLICK_BLOCK) {
            return;
        }
        EquipmentSlot hand = event.getHand();
        if (hand == null || !items.isHeart(event.getItem())) {
            return;
        }
        event.setCancelled(true);
        useHeart(event.getPlayer(), hand);
    }

    @EventHandler(priority = EventPriority.HIGH)
    public void onInteractEntity(PlayerInteractEntityEvent event) {
        EquipmentSlot hand = event.getHand();
        Player player = event.getPlayer();
        if (!items.isHeart(player.getInventory().getItem(hand))) {
            return;
        }
        event.setCancelled(true); // blokuje np. farbowanie owiec
        if (event instanceof PlayerInteractAtEntityEvent) {
            return; // zeby nie uzyc serca dwa razy przy jednym kliknieciu
        }
        useHeart(player, hand);
    }

    private void useHeart(Player player, EquipmentSlot slot) {
        ItemStack inHand = player.getInventory().getItem(slot);
        if (!items.isHeart(inHand)) {
            return;
        }
        UUID uuid = player.getUniqueId();
        int current = manager.getHearts(uuid);
        if (current >= HeartManager.MAX_HEARTS) {
            player.sendMessage(Text.color("&cMasz juz maksymalna liczbe serc (" + HeartManager.MAX_HEARTS + ")."));
            return; // przedmiot nie jest zuzywany
        }

        // Zuzycie dokladnie jednego serca.
        if (inHand.getAmount() <= 1) {
            player.getInventory().setItem(slot, null);
        } else {
            inHand.setAmount(inHand.getAmount() - 1);
        }

        manager.setHearts(uuid, current + 1);
        manager.applyMaxHealth(player);
        player.sendMessage(Text.color("&aZyskales dodatkowe serce! &7(" + (current + 1) + "/" + HeartManager.MAX_HEARTS + ")"));

        playAnimation(player, slot);
    }

    /**
     * Animacja jak przy Totemie, ale z modelem serca.
     * Klient pokazuje w animacji przedmiot z reki, ktory ma komponent death_protection.
     * Dlatego graczowi wysylamy TYLKO PO STRONIE KLIENTA falszywa kopie serca z tym komponentem,
     * a prawdziwy przedmiot go nie ma - serce nigdy nie chroni przed smiercia.
     */
    private void playAnimation(Player player, EquipmentSlot slot) {
        ItemStack fake = items.create(1);
        fake.setData(DataComponentTypes.DEATH_PROTECTION, DeathProtection.deathProtection().build());

        player.sendEquipmentChange(player, slot, fake);
        player.playEffect(EntityEffect.TOTEM_RESURRECT);
        player.playSound(player.getLocation(), Sound.ITEM_TOTEM_USE, 1.0f, 1.0f);

        Bukkit.getScheduler().runTaskLater(plugin, () -> {
            if (!player.isOnline()) {
                return;
            }
            ItemStack real = player.getInventory().getItem(slot);
            player.sendEquipmentChange(player, slot, real == null ? new ItemStack(Material.AIR) : real);
            player.updateInventory();
        }, 3L);
    }

    // ------------------------------------------------------------ bezpieczenstwo przedmiotu

    /** Serce nie moze byc skladnikiem craftingu (RED_DYE wystepuje w recepturach). */
    @EventHandler
    public void onPrepareCraft(PrepareItemCraftEvent event) {
        for (ItemStack stack : event.getInventory().getMatrix()) {
            if (items.isHeart(stack)) {
                event.getInventory().setResult(null);
                return;
            }
        }
    }

    /** Dispenser nie moze uzyc serca jako barwnika. */
    @EventHandler
    public void onDispense(BlockDispenseEvent event) {
        if (items.isHeart(event.getItem())) {
            event.setCancelled(true);
        }
    }
}

/** /give-serce (1-64) */
final class GiveHeartCommand implements CommandExecutor, TabCompleter {

    public static final String PERMISSION = "banshels.giveserce";

    private final HeartItem items;

    public GiveHeartCommand(HeartItem items) {
        this.items = items;
    }

    @Override
    public boolean onCommand(CommandSender sender, Command command, String label, String[] args) {
        if (!sender.hasPermission(PERMISSION)) {
            sender.sendMessage(Text.color("&cNie masz uprawnien do tej komendy."));
            return true;
        }
        if (!(sender instanceof Player player)) {
            sender.sendMessage(Text.color("&cTa komenda jest dostepna tylko dla graczy."));
            return true;
        }
        if (args.length != 1) {
            sender.sendMessage(Text.color("&cUzycie: /give-serce <1-64>"));
            return true;
        }

        int amount;
        try {
            amount = Integer.parseInt(args[0]);
        } catch (NumberFormatException e) {
            sender.sendMessage(Text.color("&cPodaj liczbe od 1 do 64."));
            return true;
        }
        if (amount < 1 || amount > 64) {
            sender.sendMessage(Text.color("&cLiczba musi byc z zakresu 1-64."));
            return true;
        }

        Map<Integer, ItemStack> leftover = player.getInventory().addItem(items.create(amount));
        for (ItemStack stack : leftover.values()) {
            player.getWorld().dropItemNaturally(player.getLocation(), stack);
        }
        player.sendMessage(Text.color("&aOtrzymales serca: &4" + amount));
        return true;
    }

    @Override
    public List<String> onTabComplete(CommandSender sender, Command command, String alias, String[] args) {
        List<String> result = new ArrayList<>();
        if (args.length == 1 && sender.hasPermission(PERMISSION)) {
            for (String s : List.of("1", "5", "10", "32", "64")) {
                if (s.startsWith(args[0])) {
                    result.add(s);
                }
            }
        }
        return result;
    }
}

/** /wyplacserce - zamienia jedno serce gracza na przedmiot (nie schodzi ponizej 10 serc). */
final class WithdrawHeartCommand implements CommandExecutor {

    private final HeartManager manager;
    private final HeartItem items;

    WithdrawHeartCommand(HeartManager manager, HeartItem items) {
        this.manager = manager;
        this.items = items;
    }

    @Override
    public boolean onCommand(CommandSender sender, Command command, String label, String[] args) {
        if (!(sender instanceof Player player)) {
            sender.sendMessage(Text.color("&cTa komenda jest dostepna tylko dla graczy."));
            return true;
        }

        UUID uuid = player.getUniqueId();
        int current = manager.getHearts(uuid);
        if (current <= HeartManager.MIN_WITHDRAW_HEARTS) {
            player.sendMessage(Text.color("&cNie mozna wyplacac ponizej " + HeartManager.MIN_WITHDRAW_HEARTS + " serc."));
            return true;
        }

        manager.setHearts(uuid, current - 1);
        manager.applyMaxHealth(player);

        Map<Integer, ItemStack> leftover = player.getInventory().addItem(items.create(1));
        for (ItemStack stack : leftover.values()) {
            player.getWorld().dropItemNaturally(player.getLocation(), stack);
        }
        player.sendMessage(Text.color("&aWyplaciles serce"));
        return true;
    }
}

/** Ksiega Odrodzenia - ksiazka do pisania z oznaczeniem w PersistentDataContainer. */
final class ReviveBook {

    private final NamespacedKey key;

    ReviveBook(BansheLS plugin) {
        this.key = new NamespacedKey(plugin, "revive_book");
    }

    ItemStack create() {
        ItemStack item = new ItemStack(Material.WRITABLE_BOOK);
        ItemMeta meta = item.getItemMeta();
        meta.displayName(Text.item("&6Ksiega Odrodzenia"));
        meta.lore(List.of(
                Text.color("&7&oPrzywraca do życia osobę,"),
                Text.color("&7&októra utraciła wszystkie serca!")));
        meta.getPersistentDataContainer().set(key, PersistentDataType.BYTE, (byte) 1);
        item.setItemMeta(meta);
        return item;
    }

    boolean isBook(ItemStack item) {
        if (item == null || item.getType() != Material.WRITABLE_BOOK || !item.hasItemMeta()) {
            return false;
        }
        Byte marker = item.getItemMeta().getPersistentDataContainer().get(key, PersistentDataType.BYTE);
        return marker != null && marker == (byte) 1;
    }

    /** Zwraca nick wpisany na pierwszej stronie (pierwsze slowo) albo pusty tekst. */
    String readName(ItemStack item) {
        if (item == null || !(item.getItemMeta() instanceof BookMeta bookMeta)) {
            return "";
        }
        if (bookMeta.getPageCount() < 1) {
            return "";
        }
        String page = bookMeta.getPage(1).replaceAll("§.", "").strip();
        if (page.isEmpty()) {
            return "";
        }
        return page.split("\\s+")[0];
    }
}

/** /give-ksiegaodrodzenia - tylko dla OP. */
final class GiveReviveBookCommand implements CommandExecutor {

    static final String PERMISSION = "banshels.giveksiegaodrodzenia";

    private final ReviveBook book;

    GiveReviveBookCommand(ReviveBook book) {
        this.book = book;
    }

    @Override
    public boolean onCommand(CommandSender sender, Command command, String label, String[] args) {
        if (!sender.hasPermission(PERMISSION)) {
            sender.sendMessage(Text.color("&cNie masz uprawnien do tej komendy."));
            return true;
        }
        if (!(sender instanceof Player player)) {
            sender.sendMessage(Text.color("&cTa komenda jest dostepna tylko dla graczy."));
            return true;
        }
        Map<Integer, ItemStack> leftover = player.getInventory().addItem(book.create());
        for (ItemStack stack : leftover.values()) {
            player.getWorld().dropItemNaturally(player.getLocation(), stack);
        }
        player.sendMessage(Text.color("&aOtrzymales Ksiege Odrodzenia."));
        return true;
    }
}

/** Obsluga uzycia Ksiegi Odrodzenia i animacji. */
final class ReviveListener implements Listener {

    private final BansheLS plugin;
    private final HeartManager manager;
    private final ReviveBook book;

    ReviveListener(BansheLS plugin, HeartManager manager, ReviveBook book) {
        this.plugin = plugin;
        this.manager = manager;
        this.book = book;
    }

    /**
     * Pusta ksiega albo klikniecie z SHIFTEM otwiera normalny edytor (zeby mozna bylo wpisac/poprawic nick).
     * Ksiega z wpisanym nickiem + zwykly prawy przycisk = odrodzenie.
     */
    @EventHandler(priority = EventPriority.HIGH)
    public void onInteract(PlayerInteractEvent event) {
        Action action = event.getAction();
        if (action != Action.RIGHT_CLICK_AIR && action != Action.RIGHT_CLICK_BLOCK) {
            return;
        }
        EquipmentSlot hand = event.getHand();
        ItemStack item = event.getItem();
        if (hand == null || !book.isBook(item)) {
            return;
        }
        Player player = event.getPlayer();
        if (player.isSneaking()) {
            return;
        }
        String name = book.readName(item);
        if (name.isEmpty()) {
            return;
        }
        event.setCancelled(true);
        revive(player, hand, name);
    }

    /** Ksiegi nie mozna podpisac (podpisana przestalaby byc ksiega do pisania). */
    @EventHandler
    public void onEditBook(PlayerEditBookEvent event) {
        if (!event.isSigning()) {
            return;
        }
        Player player = event.getPlayer();
        int slot = event.getSlot();
        ItemStack item = slot == -1 ? player.getInventory().getItemInOffHand() : player.getInventory().getItem(slot);
        if (book.isBook(item)) {
            event.setSigning(false);
            player.sendMessage(Text.color("&cKsiegi Odrodzenia nie mozna podpisac."));
        }
    }

    private OfflinePlayer findBanned(String name) {
        for (OfflinePlayer banned : Bukkit.getBannedPlayers()) {
            String bannedName = banned.getName();
            if (bannedName != null && bannedName.equalsIgnoreCase(name)) {
                return banned;
            }
        }
        return null;
    }

    private void revive(Player player, EquipmentSlot hand, String name) {
        OfflinePlayer target = findBanned(name);
        if (target == null) {
            player.sendMessage(Text.color("&cNie znaleziono zbanowanego gracza o nicku: &f" + name));
            return;
        }
        UUID uuid = target.getUniqueId();
        if (manager.getHearts(uuid) > 0) {
            player.sendMessage(Text.color("&cTen gracz nie utracil wszystkich serc."));
            return;
        }

        ItemStack inHand = player.getInventory().getItem(hand);
        if (!book.isBook(inHand)) {
            return;
        }
        if (inHand.getAmount() <= 1) {
            player.getInventory().setItem(hand, null);
        } else {
            inHand.setAmount(inHand.getAmount() - 1);
        }

        String realName = target.getName() != null ? target.getName() : name;
        Bukkit.dispatchCommand(Bukkit.getConsoleSender(), "minecraft:pardon " + realName);
        manager.setHearts(uuid, HeartManager.REVIVE_HEARTS);

        playAnimation(player, realName);
    }

    /** Ogien na ziemi -> bialy blysk + komunikat -> zielone czastki i plomienie w gore. */
    private void playAnimation(Player player, String targetName) {
        Location from = player.getLocation();
        Vector direction = from.getDirection().setY(0);
        if (direction.lengthSquared() < 0.0001) {
            direction = new Vector(0, 0, 1);
        }
        direction.normalize().multiply(3);
        final Location base = from.clone().add(direction);
        final World world = base.getWorld();
        final String message = "&c&l" + targetName + " &7zostal &a&lOdrodzony!";

        new BukkitRunnable() {
            int step = 0;

            @Override
            public void run() {
                if (step == 0) {
                    world.playSound(base, Sound.ITEM_FIRECHARGE_USE, 1.5f, 0.6f);
                }

                if (step < 20) {
                    double radius = 0.8 + step * 0.2;
                    for (int i = 0; i < 24; i++) {
                        double angle = ThreadLocalRandom.current().nextDouble() * Math.PI * 2;
                        double r = Math.sqrt(ThreadLocalRandom.current().nextDouble()) * radius;
                        world.spawnParticle(Particle.FLAME,
                                base.getX() + Math.cos(angle) * r, base.getY() + 0.1, base.getZ() + Math.sin(angle) * r,
                                1, 0, 0, 0, 0.01);
                    }
                    if (step % 5 == 0) {
                        world.playSound(base, Sound.BLOCK_FIRE_AMBIENT, 1.0f, 1.0f);
                    }
                } else if (step == 20) {
                    Location center = base.clone().add(0, 1, 0);
                    world.spawnParticle(Particle.EXPLOSION_EMITTER, center, 1);
                    world.spawnParticle(Particle.END_ROD, center, 120, 0.5, 1.2, 0.5, 0.25);
                    world.spawnParticle(Particle.CLOUD, center, 40, 0.8, 0.2, 0.8, 0.05);
                    world.playSound(base, Sound.ENTITY_LIGHTNING_BOLT_THUNDER, 1.5f, 1.2f);
                    world.playSound(base, Sound.ITEM_TOTEM_USE, 1.5f, 1.0f);
                    Bukkit.broadcast(Text.color(message));
                } else if (step < 40) {
                    double height = (step - 20) * 0.18;
                    double spin = step * 0.6;
                    for (int i = 0; i < 3; i++) {
                        double a = spin + i * 2.094;
                        world.spawnParticle(Particle.FLAME,
                                base.getX() + Math.cos(a) * 0.6, base.getY() + height, base.getZ() + Math.sin(a) * 0.6,
                                2, 0.05, 0.05, 0.05, 0.01);
                    }
                    world.spawnParticle(Particle.HAPPY_VILLAGER, base.getX(), base.getY() + 1.2, base.getZ(), 10, 0.4, 1.2, 0.4, 0);
                    world.spawnParticle(Particle.END_ROD, base.getX(), base.getY() + height, base.getZ(), 2, 0.1, 0.1, 0.1, 0.02);
                } else {
                    world.playSound(base, Sound.ENTITY_PLAYER_LEVELUP, 1.5f, 1.0f);
                    cancel();
                    return;
                }
                step++;
            }
        }.runTaskTimer(plugin, 0L, 2L);
    }
}
