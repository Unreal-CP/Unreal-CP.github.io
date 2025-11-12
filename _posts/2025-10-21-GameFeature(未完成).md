# GameFeature_Action
## ApplyFrontendPerfSettingsAction
`(Source\LyraGame\UI\Frontend)`
主要用于跟我们的游戏设置交互.开启和关闭前端性能限制的bool值
``` cpp
/**
 * GameFeatureAction responsible for telling the user settings to apply frontend/menu specific performance settings
 * 游戏功能操作：负责向用户告知应应用前端/菜单特定的性能设置。
 */
UCLASS(MinimalAPI, meta = (DisplayName = "Use Frontend Perf Settings"))
class UApplyFrontendPerfSettingsAction final : public UGameFeatureAction
{
	GENERATED_BODY()

public:
	//~UGameFeatureAction interface
	virtual void OnGameFeatureActivating(FGameFeatureActivatingContext& Context) override;
	virtual void OnGameFeatureDeactivating(FGameFeatureDeactivatingContext& Context) override;
	//~End of UGameFeatureAction interface

private:
	static int32 ApplicationCounter;
};
```
``` cpp
		ULyraSettingsLocal::Get()->SetShouldUseFrontendPerformanceSettings(true);
		ULyraSettingsLocal::Get()->SetShouldUseFrontendPerformanceSettings(false);
```

``` cpp
void ULyraSettingsLocal::SetShouldUseFrontendPerformanceSettings(bool bInFrontEnd)
{
	bInFrontEndForPerformancePurposes = bInFrontEnd;
	UpdateEffectiveFrameRateLimit();
}
void ULyraSettingsLocal::UpdateEffectiveFrameRateLimit()
{
	// DS服务器不需要进行该项设置
	if (!IsRunningDedicatedServer())
	{
		// 设置最大帧率
		SetFrameRateLimitCVar(GetEffectiveFrameRateLimit());
	}
}

float ULyraSettingsLocal::GetEffectiveFrameRateLimit()
{
	const ULyraPlatformSpecificRenderingSettings* PlatformSettings = ULyraPlatformSpecificRenderingSettings::Get();

#if WITH_EDITOR
	if (GIsEditor && !CVarApplyFrameRateSettingsInPIE.GetValueOnGameThread())
	{
		return Super::GetEffectiveFrameRateLimit();
	}
#endif

	if (PlatformSettings->FramePacingMode == ELyraFramePacingMode::ConsoleStyle)
	{
		return 0.0f;
	}

	float EffectiveFrameRateLimit = Super::GetEffectiveFrameRateLimit();

	if (ShouldUseFrontendPerformanceSettings())
	{
		// 选择两者的最小值
		EffectiveFrameRateLimit = CombineFrameRateLimits(EffectiveFrameRateLimit, FrameRateLimit_InMenu);
	}

	if (PlatformSettings->FramePacingMode == ELyraFramePacingMode::DesktopStyle)
	{
		// 是否使用电池
		if (FPlatformMisc::IsRunningOnBattery())
		{
			EffectiveFrameRateLimit = CombineFrameRateLimits(EffectiveFrameRateLimit, FrameRateLimit_OnBattery);
		}
		// 是否是在后台
		if (FSlateApplication::IsInitialized() && !FSlateApplication::Get().IsActive())
		{
			EffectiveFrameRateLimit = CombineFrameRateLimits(EffectiveFrameRateLimit, FrameRateLimit_WhenBackgrounded);
		}
	}

	return EffectiveFrameRateLimit;
}
```