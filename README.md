# Don't Let Your Staging Server Die: Separate Task Scheduling in Laravel

![Don't Let Your Staging Server Die: Separate Task Scheduling in Laravel](assets/poster.jpg)

Prevent staging server overload by separating schedules.

```php
final class Kernel extends ConsoleKernel
{
    protected function schedule(Schedule $schedule): void
    {
        $this->scheduleCommon($schedule);

        if ($this->app->environment('production')) {
            $this->scheduleProduction($schedule);
        }

        if ($this->app->environment('staging')) {
            $this->scheduleStaging($schedule);
        }
    }
}
```

* **Common**: runs in all environments
* **Production**: heavy, frequent jobs
* **Staging**: same jobs, less often

Result: staging CPU drops 100% → 15-20%, memory stable, tests smooth.

## 📎 Read Full

[Don't Let Your Staging Server Die: Separate Task Scheduling in Laravel](https://dev.to/tegos/)