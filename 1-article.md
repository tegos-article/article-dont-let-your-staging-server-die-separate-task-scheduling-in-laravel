If you're running Laravel in production, you probably use task scheduling.
It's one of those features that just works - until it stops. Especially on your staging server.

## The Problem

In real-world apps, you must have a staging environment. It's your safe playground to test changes before they hit
production.
I run a simple one on a Digital Ocean droplet just 2 vCPUs, nothing fancy.
It handles my Laravel 10 app like a champ... until one day, it didn't. The server just froze.
SSHing in and firing up htop revealed both CPU cores maxed out and memory nearly full:

```
   1[||||||||||||||||||100.0%]   
   2[||||||||||||||||||100.0%]     
Mem[||||||||||||||| 3.2G/4.0G]
```

Oof. Both cores pegged at 100%, and memory was gasping at 3.2G out of 4G. Not good.

## Why It Matters

The guilty party? A quick ps aux and log dive pointed straight to the scheduler.
Heavy scheduled commands that make sense in production but kill a smaller staging server:
price exports every 2h, supplier syncs with thousands of products, and stat metric collection processing large datasets.

These tasks are necessary in production with proper resources. But staging doesn't need them at the same frequency - or
at all.

## The Solution

The fix is simple: separate your `schedule()` method based on environment. Don't run the same schedule everywhere.

I'm on Laravel 10, so I tweak the app/Console/Kernel.php. (Heads up: In Laravel 11+, shift to routes/console.php)

```php
final class Kernel extends ConsoleKernel
{
    protected function schedule(Schedule $schedule): void
    {
        $this->scheduleCommon($schedule);

        if ($this->app->environment(AppEnvironmentEnum::PRODUCTION->value)) {
            $this->scheduleProduction($schedule);
        }

        if ($this->app->environment(AppEnvironmentEnum::STAGING->value)) {
            // Less frequent or reduced workload for staging
            $this->scheduleStaging($schedule);
        }
    }
}
```

## How to Structure It

### Common Schedule

Tasks that should run everywhere: cleanup, pruning, basic maintenance.

```php
private function scheduleCommon(Schedule $schedule): void
{
    // Keep DB lean everywhere
    $schedule->command(PruneCommand::class, ['--model' => [CartItem::class]])->hourly();
    $schedule->command(PruneCommand::class, ['--model' => [OrderItemNotification::class]])->daily();
    
    // Clear expired tokens
    $schedule->command(PruneExpired::class)->dailyAt('03:00');
    
    // Basic notifications
    $schedule->command(TelegramNotificationSendCommand::class)->hourly()->withoutOverlapping();
    $schedule->command(UserSendConfirmationNotificationCommand::class)->hourly();
}
```

### Production Schedule

Heavy operations that need frequent execution and have proper resources.

```php
private function scheduleProduction(Schedule $schedule): void
{
    // Search index sync
    $schedule->command(SearchIndexSyncDiff::class)->dailyAt('05:00');
    
    // Auto price exports - multiple times per day
    $schedule->command(PriceExportAutoDispatchCommand::class)->cron('15 6,8,10,12,14,16,18 * * *');
    
    // Supplier sync - runs 7 times per day
    $schedule->command(TmSyncPriceExportProduct::class)->cron('1 6,8,10,12,14,16,18 * * *');
    
    // Product availability updates
    $schedule->command(ProductAvailabilityUpdateCommand::class)->hourlyAt(15)->between('08:00', '19:00');
}
```

### Staging Schedule

Same commands, but less frequent. Only what you actually need for testing.

```php
private function scheduleStaging(Schedule $schedule): void
{
    // Price exports - once per day is enough
    $schedule->command(PriceExportAutoDispatchCommand::class)->cron('15 6 * * *');
    
    // Supplier sync - once per day
    $schedule->command(TmSyncPriceExportProduct::class)->cron('1 6 * * *');
    
    // Test suspended orders more frequently for debugging
    $schedule->command(OrderProcessSuspendedCommand::class)->everyMinute()->between('08:00', '19:00')->withoutOverlapping();
}
```

## The Results

After separating the workloads, my staging cores chilled under 20% and memory remained stable like 1.5GB.
The server stayed responsive, and I can still run scheduled tasks whenever I need to. Production continued operating at
full capacity without any impact.
Overall, staging is smooth again, tests run quickly, and the setup scales easily as more environments are added.

| Env              | CPU Usage | Memory Usage |
|------------------|-----------|--------------|
| Staging (before) | 100%      | 3.2G / 4G    |
| Staging (after)  | 15-20%    | 1.5G / 4G    |

## Conclusions

Separate task schedules by environment from day one, and let staging take it easy.
Heavy jobs should run less often and keep only essential maintenance routines.
Copying production schedules blindly to staging is a fast track to chaos - test servers usually have less CPU and
memory.

After these changes, staging CPU drops to 15-20%, tests run smoothly, and deployments feel confident. Production stays
strong and untouched.
This setup scales easily - just add environment checks for new tiers.

Your staging environment should mirror production functionally, not identically. Adjust task frequencies to match your
server resources.
Your infrastructure will thank you.