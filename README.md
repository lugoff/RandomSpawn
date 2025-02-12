# RandomSpawn

package com.lugoff.randomSpawn;

    private int maxPosX;
    private int maxNegX;
    private int maxPosZ;
    private int maxNegZ;
    private int maxY;
    private boolean debugMode;
    private boolean showRespawnLocation;
    private boolean enableBedRespawn;
    private Map<Player, Location> respawnLocations = new HashMap<>();

    @Override
    public void onEnable() {
        saveDefaultConfig();
        loadConfig();
        getServer().getPluginManager().registerEvents(this, this);
        getLogger().info("RandomSpawn enabled! Author: lugoff");
    }

    @Override
    public void onDisable() {
        getLogger().info("RandomSpawn disabled!");
    }

    @EventHandler
    public void onPlayerDeath(PlayerDeathEvent event) {
        Player player = event.getEntity();
        Location respawnLocation = getRespawnLocation(player);
        respawnLocations.put(player, respawnLocation);

        if (debugMode) {
            getLogger().info("Generated respawn location for player: X=" + respawnLocation.getBlockX() +
                    ", Y=" + respawnLocation.getBlockY() +
                    ", Z=" + respawnLocation.getBlockZ());
        }
    }

    @EventHandler
    public void onPlayerRespawn(PlayerRespawnEvent event) {
        Player player = event.getPlayer();
        Location respawnLocation = respawnLocations.get(player);

        if (respawnLocation != null) {
            if (debugMode) {
                getLogger().info("Setting respawn location for player: X=" + respawnLocation.getBlockX() +
                        ", Y=" + respawnLocation.getBlockY() +
                        ", Z=" + respawnLocation.getBlockZ());
            }

            event.setRespawnLocation(respawnLocation);

            Bukkit.getScheduler().runTaskLater(this, () -> {
                if (debugMode) {
                    getLogger().info("Teleporting player to: X=" + respawnLocation.getBlockX() +
                            ", Y=" + respawnLocation.getBlockY() +
                            ", Z=" + respawnLocation.getBlockZ());
                }

                player.teleport(respawnLocation);

                if (showRespawnLocation) {
                    player.sendMessage("You have been respawned at a random location!");
                    player.sendMessage("Respawned at X: " + respawnLocation.getBlockX() +
                            ", Y: " + respawnLocation.getBlockY() +
                            ", Z: " + respawnLocation.getBlockZ());
                }

                respawnLocations.remove(player);
            }, 1L); // Slight delay to ensure the player is fully respawned
        }
    }

    @Override
    public boolean onCommand(CommandSender sender, Command command, String label, String[] args) {
        if (command.getName().equalsIgnoreCase("randomrespawn")) {
            sender.sendMessage("The author of this plugin is lugoff");
            return true;
        }
        return false;
    }

    private Location getRespawnLocation(Player player) {
        if (enableBedRespawn) {
            Location bedLocation = player.getBedSpawnLocation();
            if (bedLocation != null) {
                return bedLocation;
            }
        }
        World world = player.getWorld();
        return getRandomLocation(world);
    }

    private Location getRandomLocation(World world) {
        Random random = new Random();
        Location randomLocation;
        int attempts = 0;
        final int MAX_ATTEMPTS = 20; // Limit attempts to avoid infinite loops

        do {
            int x = generateCoordinate(random, maxNegX, maxPosX);
            int y = maxY; // Use configured maxY
            int z = generateCoordinate(random, maxNegZ, maxPosZ);

            randomLocation = new Location(world, x, y, z);
            int groundY = world.getHighestBlockYAt(randomLocation);
            randomLocation.setY(groundY + 1); // Get ground level and set it

            attempts++;
            if (attempts > MAX_ATTEMPTS) {
                getLogger().warning("Max attempts reached to find safe spawn. Using 0, 100, 0");
                return new Location(world, 0, 100, 0); // Default location if all attempts fail
            }

            if (debugMode) {
                getLogger().info("Attempt " + attempts + " - Generated coordinates: X=" + x + ", Z=" + z + ", groundY=" + groundY); // Log to console
            }
        } while (!isLocationSafe(randomLocation));

        if (debugMode) {
            getLogger().info("Safe location found at X=" + randomLocation.getBlockX() + ", Z=" + randomLocation.getBlockZ()); // Log safe location
        }
        return randomLocation;
    }

    private int generateCoordinate(Random random, int negativeLimit, int positiveLimit) {
        return random.nextInt(positiveLimit - negativeLimit + 1) + negativeLimit;
    }

    private boolean isLocationSafe(Location location) {
        // Check if the block at the location and the block above it are air
        Block block = location.getBlock();
        Location aboveLocation = location.clone().add(0, 1, 0); // Create a NEW Location
        Block above = aboveLocation.getBlock(); // Get the block above

        Location belowLocation = location.clone().add(0, -2, 0); // Create a NEW Location
        Block below = belowLocation.getBlock(); // Get the block below

        return block.isPassable() && above.isPassable() && !below.isPassable(); // Must be able to walk and not in a wall.
    }

    private void loadConfig() {
        FileConfiguration config = getConfig();
        maxPosX = config.getInt("maxPosX", 1000);
        maxNegX = config.getInt("maxNegX", -1000);
        maxPosZ = config.getInt("maxPosZ", 1000);
        maxNegZ = config.getInt("maxNegZ", -1000);
        maxY = config.getInt("maxY", 128);
        debugMode = config.getBoolean("debugMode", false);
        showRespawnLocation = config.getBoolean("showRespawnLocation", true);
        enableBedRespawn = config.getBoolean("enableBedRespawn", true);
    }
}
