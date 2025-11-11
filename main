package com.swill.cheats.silentaim;

import net.minecraft.client.Minecraft;
import net.minecraft.network.chat.Component;
import net.minecraft.world.entity.LivingEntity;
import net.minecraft.world.entity.animal.Animal;
import net.minecraft.world.entity.animal.WaterAnimal;
import net.minecraft.world.entity.monster.Monster;
import net.minecraft.world.entity.npc.Npc;
import net.minecraft.world.entity.player.Player;
import net.minecraft.world.phys.Vec3;
import net.minecraftforge.api.distmarker.Dist;
import net.minecraftforge.common.MinecraftForge;
import net.minecraftforge.event.TickEvent;
import net.minecraftforge.event.entity.EntityJoinLevelEvent;
import net.minecraftforge.eventbus.api.SubscribeEvent;
import net.minecraftforge.fml.common.Mod;
import net.minecraftforge.fml.loading.FMLEnvironment;

import java.lang.reflect.Method;
import java.util.ArrayList;
import java.util.List;

@Mod("swill_silentaim")
public class SilentAimMod {

    private static boolean enabled = true;
    private static final List<Object> trackedBullets = new ArrayList<>();
    private static long lastToggleTime = 0;

    // Настройки целей
    private static boolean targetPlayers = true;
    private static boolean targetMonsters = true;
    private static boolean targetAnimals = true;
    private static boolean targetNPCs = false;

    // Состояния клавиш для анти-дребезга
    private static boolean lastVKeyState = false;
    private static boolean lastF1KeyState = false;
    private static boolean lastF2KeyState = false;
    private static boolean lastOKeyState = false;
    private static boolean lastF4KeyState = false;
    private static boolean lastF5KeyState = false;

    // Дебаг информация
    private static int debugCounter = 0;
    private static boolean showDebugInfo = false;

    // Флаг для определения клиентской стороны
    private static final boolean IS_CLIENT = FMLEnvironment.dist == Dist.CLIENT;

    public SilentAimMod() {
        if (IS_CLIENT) {
            MinecraftForge.EVENT_BUS.register(this);
            sendChatMessage("§aSWILL Silent Aim loaded - Server Compatible");
            printControls();
        }
    }

    private void printControls() {
        sendChatMessage("§6=== SWILL Silent Aim Controls ===");
        sendChatMessage("§eV §7- Toggle Silent Aim");
        sendChatMessage("§eF1 §7- Target Players: " + (targetPlayers ? "§aON" : "§cOFF"));
        sendChatMessage("§eF2 §7- Target Monsters: " + (targetMonsters ? "§aON" : "§cOFF"));
        sendChatMessage("§eO §7- Target Animals: " + (targetAnimals ? "§aON" : "§cOFF"));
        sendChatMessage("§eF4 §7- Target NPCs: " + (targetNPCs ? "§aON" : "§cOFF"));
        sendChatMessage("§eF5 §7- Toggle Debug Info");
        sendChatMessage("§6=== Works on any TacZ server ===");
        sendChatMessage("§6================================");
    }

    private void sendChatMessage(String message) {
        if (IS_CLIENT && Minecraft.getInstance().player != null) {
            Minecraft.getInstance().player.displayClientMessage(Component.literal(message), false);
        } else {
            System.out.println(message.replace("§", ""));
        }
    }

    @SubscribeEvent
    public void onEntitySpawn(EntityJoinLevelEvent event) {
        // Работаем только на клиенте и только если включено
        if (!IS_CLIENT || !enabled) return;

        try {
            // Безопасная проверка через рефлексию - не зависит от серверных классов
            Class<?> entityClass = event.getEntity().getClass();
            String className = entityClass.getName();
            
            // Проверяем является ли entity пулей TacZ (работает на любом сервере с TacZ)
            if (className.equals("com.tacz.guns.entity.EntityKineticBullet")) {
                Object bullet = event.getEntity();
                Player shooter = getPlayer();
                
                if (shooter != null && isOwnBullet(bullet, shooter)) {
                    trackedBullets.add(bullet);
                    if (showDebugInfo) {
                        sendChatMessage("§a[SWILL] Bullet tracked");
                    }
                    correctBulletTrajectory(bullet, shooter);
                }
            }
        } catch (Exception e) {
            if (showDebugInfo) {
                sendChatMessage("§c[SWILL] Spawn error: " + e.getMessage());
            }
        }
    }

    private Player getPlayer() {
        return IS_CLIENT ? Minecraft.getInstance().player : null;
    }

    @SubscribeEvent
    public void onClientTick(TickEvent.ClientTickEvent event) {
        // Работаем только на клиенте
        if (!IS_CLIENT || event.phase != TickEvent.Phase.END) return;

        // Очистка удаленных пуль
        trackedBullets.removeIf(bullet -> {
            try {
                Method isRemoved = bullet.getClass().getMethod("isRemoved");
                return (Boolean) isRemoved.invoke(bullet);
            } catch (Exception e) {
                return true;
            }
        });

        // Коррекция активных пуль
        if (enabled) {
            Player shooter = getPlayer();
            if (shooter != null) {
                for (Object bullet : trackedBullets) {
                    if (isOwnBullet(bullet, shooter)) {
                        correctBulletTrajectory(bullet, shooter);
                    }
                }
            }

            // Дебаг информация каждые 10 секунд
            debugCounter++;
            if (debugCounter >= 200 && showDebugInfo) {
                printDebugInfo();
                debugCounter = 0;
            }
        }

        // Обработка горячих клавиш
        handleToggleKey();
        handleTargetTypeKeys();
    }

    private void printDebugInfo() {
        Player player = getPlayer();
        if (player == null) return;

        try {
            // Используем рефлексию для доступа к серверным методам
            Method getLevel = player.getClass().getMethod("m_9236_"); // level()
            Object level = getLevel.invoke(player);
            
            Method getEntities = level.getClass().getMethod("m_45976_", Class.class, net.minecraft.world.phys.AABB.class, java.util.function.Predicate.class);
            
            // Поиск животных
            List<LivingEntity> animals = (List<LivingEntity>) getEntities.invoke(level, 
                LivingEntity.class,
                player.getBoundingBox().inflate(50.0),
                (java.util.function.Predicate<LivingEntity>) entity -> 
                    entity instanceof Animal || entity instanceof WaterAnimal
            );

            // Поиск монстров
            List<LivingEntity> monsters = (List<LivingEntity>) getEntities.invoke(level, 
                LivingEntity.class,
                player.getBoundingBox().inflate(50.0),
                (java.util.function.Predicate<LivingEntity>) entity -> 
                    entity instanceof Monster
            );

            sendChatMessage("§6[SWILL] Nearby - Animals: §a" + animals.size() + "§6, Monsters: §c" + monsters.size() + "§6, Bullets: §e" + trackedBullets.size());
        } catch (Exception e) {
            sendChatMessage("§c[SWILL] Debug error");
        }
    }

    private boolean isOwnBullet(Object bullet, Player player) {
        try {
            Method getOwner = bullet.getClass().getMethod("getOwner");
            Object owner = getOwner.invoke(bullet);
            return owner == player;
        } catch (Exception e) {
            return false;
        }
    }

    private void correctBulletTrajectory(Object bullet, Player shooter) {
        try {
            LivingEntity target = findBestTarget(shooter);
            if (target != null && isValidTarget(target, shooter)) {
                if (showDebugInfo) {
                    String targetName = getTargetName(target);
                    sendChatMessage("§a[SWILL] Aiming at: §e" + targetName);
                }
                adjustBulletToTarget(bullet, target);
            }
        } catch (Exception e) {
            if (showDebugInfo) {
                sendChatMessage("§c[SWILL] Aim error");
            }
        }
    }

    private String getTargetName(LivingEntity entity) {
        if (entity instanceof Player) {
            return "Player";
        } else if (entity instanceof Monster) {
            return "Monster";
        } else if (entity instanceof Animal) {
            return "Animal";
        } else if (entity instanceof WaterAnimal) {
            return "Water Animal";
        } else if (entity instanceof Npc) {
            return "NPC";
        } else {
            try {
                return entity.getType().getDescription().getString();
            } catch (Exception e) {
                return "Unknown";
            }
        }
    }

    private LivingEntity findBestTarget(Player shooter) {
        try {
            Vec3 viewVec = shooter.getViewVector(1.0F);
            Vec3 eyePos = shooter.getEyePosition(1.0F);

            // Используем рефлексию для поиска entity на сервере
            Method getLevel = shooter.getClass().getMethod("m_9236_"); // level()
            Object level = getLevel.invoke(shooter);
            
            Method getEntities = level.getClass().getMethod("m_45976_", Class.class, net.minecraft.world.phys.AABB.class, java.util.function.Predicate.class);
            
            List<LivingEntity> potentialTargets = (List<LivingEntity>) getEntities.invoke(level, 
                LivingEntity.class,
                shooter.getBoundingBox().inflate(100.0),
                (java.util.function.Predicate<LivingEntity>) entity -> isValidTarget(entity, shooter)
            );

            if (potentialTargets.isEmpty()) {
                return null;
            }

            return potentialTargets.stream()
                .min((e1, e2) -> {
                    double angle1 = calculateAngleToTarget(eyePos, viewVec, e1);
                    double angle2 = calculateAngleToTarget(eyePos, viewVec, e2);

                    // Приоритет целей по типам
                    int priority1 = getTargetPriority(e1);
                    int priority2 = getTargetPriority(e2);
                    
                    if (priority1 != priority2) {
                        return Integer.compare(priority2, priority1);
                    }

                    return Double.compare(angle1, angle2);
                })
                .orElse(null);
        } catch (Exception e) {
            return null;
        }
    }

    private int getTargetPriority(LivingEntity entity) {
        if (entity instanceof Player) return 4;
        if (entity instanceof Monster) return 3;  
        if (entity instanceof Animal) return 2;
        if (entity instanceof WaterAnimal) return 2;
        if (entity instanceof Npc) return 1;
        return 0;
    }

    private boolean isValidTarget(LivingEntity entity, Player shooter) {
        if (entity == shooter) return false;
        if (!entity.isAlive()) return false;
        if (entity.isSpectator()) return false;
        
        // Проверка дистанции
        double distance = shooter.distanceTo(entity);
        if (distance > 100.0) return false;

        // Проверяем тип цели согласно настройкам
        if (entity instanceof Player) {
            return targetPlayers && !entity.isAlliedTo(shooter);
        }
        else if (entity instanceof Monster) {
            return targetMonsters;
        }
        else if (entity instanceof Animal || entity instanceof WaterAnimal) {
            return targetAnimals;
        }
        else if (entity instanceof Npc) {
            return targetNPCs;
        }

        return targetAnimals;
    }

    private double calculateAngleToTarget(Vec3 eyePos, Vec3 viewVec, LivingEntity target) {
        try {
            Vec3 targetPos = getTargetHitPos(target);
            Vec3 toTarget = targetPos.subtract(eyePos).normalize();
            double dot = Math.max(-1.0, Math.min(1.0, viewVec.dot(toTarget)));
            return Math.toDegrees(Math.acos(dot));
        } catch (Exception e) {
            return 180.0;
        }
    }

    private Vec3 getTargetHitPos(LivingEntity target) {
        try {
            if (target instanceof Player) {
                return target.getEyePosition(1.0F).subtract(0, target.getEyeHeight() * 0.2, 0);
            } else if (target instanceof Animal) {
                return target.position().add(0, target.getBbHeight() * 0.6, 0);
            } else if (target instanceof WaterAnimal) {
                return target.position().add(0, target.getBbHeight() * 0.5, 0);
            } else {
                return target.position().add(0, target.getBbHeight() * 0.7, 0);
            }
        } catch (Exception e) {
            return target.position();
        }
    }

    private void adjustBulletToTarget(Object bullet, LivingEntity target) {
        try {
            Method positionMethod = bullet.getClass().getMethod("position");
            Method getDeltaMovement = bullet.getClass().getMethod("getDeltaMovement");
            Method setDeltaMovement = bullet.getClass().getMethod("setDeltaMovement", Vec3.class);

            Vec3 bulletPos = (Vec3) positionMethod.invoke(bullet);
            Vec3 currentVel = (Vec3) getDeltaMovement.invoke(bullet);

            if (currentVel.length() > 0.1) {
                Vec3 targetPos = getTargetHitPos(target);
                Vec3 desiredDir = targetPos.subtract(bulletPos).normalize();

                double correctionStrength = (target instanceof Animal) ? 0.9 : 0.8;
                Vec3 currentDir = currentVel.normalize();
                Vec3 newDir = currentDir.scale(1.0 - correctionStrength)
                                      .add(desiredDir.scale(correctionStrength))
                                      .normalize();

                setDeltaMovement.invoke(bullet, newDir.scale(currentVel.length()));
            }
        } catch (Exception e) {
            // Игнорируем ошибки рефлексии
        }
    }

    private void handleToggleKey() {
        if (!IS_CLIENT) return;
        
        boolean currentVKeyState = isKeyPressed(org.lwjgl.glfw.GLFW.GLFW_KEY_V);
        
        if (currentVKeyState && !lastVKeyState) {
            enabled = !enabled;
            if (enabled) {
                sendChatMessage("§a[SWILL] Silent Aim §2ENABLED");
            } else {
                sendChatMessage("§a[SWILL] Silent Aim §cDISABLED");
                trackedBullets.clear();
            }
        }
        
        lastVKeyState = currentVKeyState;
    }

    private void handleTargetTypeKeys() {
        if (!IS_CLIENT) return;

        // F1 - Игроки
        boolean currentF1KeyState = isKeyPressed(org.lwjgl.glfw.GLFW.GLFW_KEY_F1);
        if (currentF1KeyState && !lastF1KeyState) {
            targetPlayers = !targetPlayers;
            sendChatMessage("§a[SWILL] Target Players: " + (targetPlayers ? "§aON" : "§cOFF"));
        }
        lastF1KeyState = currentF1KeyState;

        // F2 - Монстры
        boolean currentF2KeyState = isKeyPressed(org.lwjgl.glfw.GLFW.GLFW_KEY_F2);
        if (currentF2KeyState && !lastF2KeyState) {
            targetMonsters = !targetMonsters;
            sendChatMessage("§a[SWILL] Target Monsters: " + (targetMonsters ? "§aON" : "§cOFF"));
        }
        lastF2KeyState = currentF2KeyState;

        // O - Животные
        boolean currentOKeyState = isKeyPressed(org.lwjgl.glfw.GLFW.GLFW_KEY_O);
        if (currentOKeyState && !lastOKeyState) {
            targetAnimals = !targetAnimals;
            sendChatMessage("§a[SWILL] Target Animals: " + (targetAnimals ? "§aON" : "§cOFF"));
        }
        lastOKeyState = currentOKeyState;

        // F4 - NPC
        boolean currentF4KeyState = isKeyPressed(org.lwjgl.glfw.GLFW.GLFW_KEY_F4);
        if (currentF4KeyState && !lastF4KeyState) {
            targetNPCs = !targetNPCs;
            sendChatMessage("§a[SWILL] Target NPCs: " + (targetNPCs ? "§aON" : "§cOFF"));
        }
        lastF4KeyState = currentF4KeyState;

        // F5 - Дебаг информация
        boolean currentF5KeyState = isKeyPressed(org.lwjgl.glfw.GLFW.GLFW_KEY_F5);
        if (currentF5KeyState && !lastF5KeyState) {
            showDebugInfo = !showDebugInfo;
            sendChatMessage("§a[SWILL] Debug Info: " + (showDebugInfo ? "§aON" : "§cOFF"));
        }
        lastF5KeyState = currentF5KeyState;
    }

    private boolean isKeyPressed(int keyCode) {
        if (!IS_CLIENT || Minecraft.getInstance().screen != null) {
            return false;
        }
        return org.lwjgl.glfw.GLFW.glfwGetKey(
                Minecraft.getInstance().getWindow().getWindow(),
                keyCode
        ) == org.lwjgl.glfw.GLFW.GLFW_PRESS;
    }

    public static boolean isEnabled() {
        return enabled;
    }

    public static boolean isTargetingPlayers() { return targetPlayers; }
    public static boolean isTargetingMonsters() { return targetMonsters; }
    public static boolean isTargetingAnimals() { return targetAnimals; }
    public static boolean isTargetingNPCs() { return targetNPCs; }
}
