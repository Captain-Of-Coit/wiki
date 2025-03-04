# Official Mod Example

Extracted from `ModPostProcessor` found in official game release build.

This will give us an idea of how mods are meant to be written for CS2.

<div class="warning">
This is of course subject to change once the official modding platform arrives. It's only here to give a start and some idea of what to expect.
</div>

### `ModPostProcessor.Resources.Mod.cs`

<details>

```csharp
#define BURST
//#define VERBOSE

using Game;
using Game.Citizens;
using Game.Modding;
using Game.Simulation;
using Unity.Burst;
using Unity.Collections;
using Unity.Entities;
using Unity.Jobs;
using Colossal.Logging;

namespace ModSample
{
    public class TestMod : IMod
    {
        public static ILog log = LogManager.GetLogger(nameof(TestMod), false);

        public void OnCreateWorld(UpdateSystem updateSystem)
        {
            log.Info(nameof(OnCreateWorld));
            updateSystem.UpdateAt<PrintPopulationSystem>(SystemUpdatePhase.GameSimulation);
            updateSystem.UpdateAt<DeltaTimePrintSystem>(SystemUpdatePhase.GameSimulation);
            updateSystem.UpdateAt<TestModSystem>(SystemUpdatePhase.GameSimulation);
        }

        public void OnDispose()
        {
            log.Info(nameof(OnDispose));
        }

        public void OnLoad()
        {
            log.Info(nameof(OnLoad));
        }
    }

    public partial class DeltaTimePrintSystem : GameSystemBase
    {
        protected override void OnCreate()
        {
            base.OnCreate();

            TestMod.log.Info($"[{nameof(DeltaTimePrintSystem)}] {nameof(OnCreate)}");
        }
        protected override void OnUpdate()
        {
            var deltaTime = SystemAPI.Time.DeltaTime;
            TestMod.log.Info($"[{nameof(DeltaTimePrintSystem)}] DeltaTime: {deltaTime}");
        }
    }

    public partial class PrintPopulationSystem : GameSystemBase
    {
        private SimulationSystem m_SimulationSystem;
        private EntityQuery m_HouseholdQuery;

        private NativeArray<int> m_ResultArray;
        protected override void OnCreate()
        {
            base.OnCreate();

            TestMod.log.Info($"[{nameof(PrintPopulationSystem)}] {nameof(OnCreate)}");

            m_SimulationSystem = World.GetOrCreateSystemManaged<SimulationSystem>();

            m_HouseholdQuery = GetEntityQuery(
                ComponentType.ReadOnly<Household>(),
                ComponentType.Exclude<TouristHousehold>(),
                ComponentType.Exclude<CommuterHousehold>(),
                ComponentType.ReadOnly<Game.Buildings.PropertyRenter>(),
                ComponentType.Exclude<Game.Common.Deleted>(),
                ComponentType.Exclude<Game.Tools.Temp>()
                );

            m_ResultArray = new NativeArray<int>(1, Allocator.Persistent);
        }
        protected override void OnUpdate()
        {
            if (m_SimulationSystem.frameIndex % 128 == 75)
            {
                TestMod.log.Info($"[{nameof(PrintPopulationSystem)}] Population: {m_ResultArray[0]}");

                var popJob = new CountPopulationJob
                {
                    m_HouseholdChunks = m_HouseholdQuery.ToArchetypeChunkArray(Allocator.TempJob),
                    m_HouseholdCitizenType = GetBufferTypeHandle<HouseholdCitizen>(true),
                    m_CommuterType = GetComponentTypeHandle<CommuterHousehold>(true),
                    m_MovingAwayType = GetComponentTypeHandle<Game.Agents.MovingAway>(true),
                    m_TouristType = GetComponentTypeHandle<TouristHousehold>(true),
                    m_HouseholdType = GetComponentTypeHandle<Household>(true),

                    m_HealthProblems = GetComponentLookup<HealthProblem>(true),
                    m_Citizens = GetComponentLookup<Citizen>(true),

                    m_Result = m_ResultArray,
                };

                Dependency = popJob.Schedule();
                CompleteDependency();
            }
        }

#if BURST
        [BurstCompile]
#endif
        public struct CountPopulationJob : IJob
        {
            [DeallocateOnJobCompletion] [ReadOnly] public NativeArray<ArchetypeChunk> m_HouseholdChunks;
            [ReadOnly] public BufferTypeHandle<HouseholdCitizen> m_HouseholdCitizenType;
            [ReadOnly] public ComponentTypeHandle<TouristHousehold> m_TouristType;
            [ReadOnly] public ComponentTypeHandle<CommuterHousehold> m_CommuterType;
            [ReadOnly] public ComponentTypeHandle<Game.Agents.MovingAway> m_MovingAwayType;
            [ReadOnly] public ComponentTypeHandle<Household> m_HouseholdType;

            [ReadOnly] public ComponentLookup<Citizen> m_Citizens;
            [ReadOnly] public ComponentLookup<HealthProblem> m_HealthProblems;

            public NativeArray<int> m_Result;

            public void Execute()
            {
#if VERBOSE
                TestMod.Log.Debug($"Start executing {nameof(CountPopulationJob)}");
#endif
                m_Result[0] = 0;

                for (int i = 0; i < m_HouseholdChunks.Length; ++i)
                {
                    ArchetypeChunk chunk = m_HouseholdChunks[i];
                    BufferAccessor<HouseholdCitizen> citizenBuffers = chunk.GetBufferAccessor(ref m_HouseholdCitizenType);
                    NativeArray<Household> households = chunk.GetNativeArray(ref m_HouseholdType);

                    if (chunk.Has(ref m_TouristType) || chunk.Has(ref m_CommuterType) || chunk.Has(ref m_MovingAwayType))
                        continue;

                    for (int j = 0; j < chunk.Count; ++j)
                    {
                        if ((households[j].m_Flags & HouseholdFlags.MovedIn) == 0)
                            continue;

                        DynamicBuffer<HouseholdCitizen> citizens = citizenBuffers[j];
                        for (int k = 0; k < citizens.Length; ++k)
                        {
                            Entity citizen = citizens[k].m_Citizen;
                            if (m_Citizens.HasComponent(citizen) && !CitizenUtils.IsDead(citizen, ref m_HealthProblems))
                                m_Result[0] += 1;
                        }
                    }
                }
#if VERBOSE
                TestMod.Log.Debug($"Finish executing {nameof(CountPopulationJob)}");
#endif
            }
        }
    }

#if BURST
    [BurstCompile]
#endif
    public partial class TestModSystem : GameSystemBase
    {
        private SimulationSystem m_SimulationSystem;
        private NativeArray<int> m_Array;

        protected override void OnCreate()
        {
            m_SimulationSystem = World.GetOrCreateSystemManaged<SimulationSystem>();
            m_Array = new NativeArray<int>(5, Allocator.Persistent);
        }
        protected override void OnUpdate()
        {
            if (m_SimulationSystem.frameIndex % 128 == 75)
            {
                TestMod.log.Info(string.Join(", ", m_Array));

                var testJob = new TestJob
                {
                    m_Array = m_Array,
                };

                Dependency = testJob.Schedule();
            }
        }

#if BURST
        [BurstCompile]
#endif
        public struct TestJob : IJob
        {
            public NativeArray<int> m_Array;

            public void Execute()
            {
#if VERBOSE
                UnityEngine.Debug.Log($"Start executing {nameof(TestJob)}");
#endif
                for (int i = 0; i < m_Array.Length; i += 1)
                {
                    m_Array[i] = m_Array[i] + i;
                }
#if VERBOSE
                UnityEngine.Debug.Log($"Finish executing {nameof(TestJob)}");
#endif
            }

#if BURST
            [BurstCompile]
#endif
            public static void WorkTime(in long start, in long current, out long duration)
            {
                duration = current - start;
            }
        }
    }
}
```
</details>

### `ModPostProcessor.Resources.Configuration.csproj`

<details>

```xml
<Project>
	<PropertyGroup>
		<TargetFramework>net472</TargetFramework>
		<RunPostBuildEvent>OnOutputUpdated</RunPostBuildEvent>
		<Configurations>Editor;Steam</Configurations>

		<UnityVersion>2022.3.7f1</UnityVersion>
		<EntityVersion>1.0.14</EntityVersion>
		<InstallationPath>C:\Program Files (x86)\Steam\steamapps\common\Cities Skylines II</InstallationPath>
		<DataPath>$(LOCALAPPDATA)\..\LocalLow\Colossal Order\Cities Skylines II</DataPath>
		<RepoPath Condition="'$(RepoPath)'==''">$(ProjectDir)..\..\..\..</RepoPath>

		<UnityModProjectPath>$(DataPath)\.cache\Modding\UnityModsProject</UnityModProjectPath>

		<ModPostProcessorPath Condition="'$(Configuration)'=='Editor'">$(RepoPath)\BeverlyHills\Assets\StreamingAssets\~Tooling~\ModPostProcessor\ModPostProcessor.exe</ModPostProcessorPath>
		<ModPostProcessorPath Condition="'$(Configuration)'=='Steam'">$(InstallationPath)\Cities2_Data\StreamingAssets\~Tooling~\ModPostProcessor\ModPostProcessor.exe</ModPostProcessorPath>

		<EntityPackagePath>$(UnityModProjectPath)\Library\PackageCache\com.unity.entities@$(EntityVersion)\Unity.Entities\SourceGenerators</EntityPackagePath>

		<ManagedDLLPath Condition="'$(Configuration)'=='Editor'">$(RepoPath)\BeverlyHills\Library\ScriptAssemblies</ManagedDLLPath>
		<ManagedDLLPath Condition="'$(Configuration)'=='Steam'">$(InstallationPath)\Cities2_Data\Managed</ManagedDLLPath>

		<UnityEnginePath>C:\Program Files\Unity\Hub\Editor\$(UnityVersion)\Editor\Data\Managed\UnityEngine</UnityEnginePath>

		<AssemblySearchPaths Condition="'$(Configuration)'=='Editor'">
			$(AssemblySearchPaths);
			$(ManagedDLLPath);
			$(UnityEnginePath);
		</AssemblySearchPaths>
		<AssemblySearchPaths Condition="'$(Configuration)'=='Steam'">
			$(AssemblySearchPaths);
			$(ManagedDLLPath);
		</AssemblySearchPaths>
	
	</PropertyGroup>

	<ItemGroup>
		<Analyzer Include="$(EntityPackagePath)\Unity.Entities.SourceGen.SystemGenerator.SystemAPI.dll" />
		<Analyzer Include="$(EntityPackagePath)\Unity.Entities.SourceGen.SystemGenerator.dll" />
		<Analyzer Include="$(EntityPackagePath)\Unity.Entities.SourceGen.SystemGenerator.SystemAPI.QueryBuilder.dll" />
		<Analyzer Include="$(EntityPackagePath)\Unity.Entities.Analyzer.dll" />
		<Analyzer Include="$(EntityPackagePath)\Unity.Entities.SourceGen.SystemGenerator.LambdaJobs.dll" />
		<Analyzer Include="$(EntityPackagePath)\Unity.Entities.SourceGen.SystemGenerator.Common.dll" />
		<Analyzer Include="$(EntityPackagePath)\Unity.Entities.SourceGen.SystemGenerator.SystemAPI.Query.dll" />
		<Analyzer Include="$(EntityPackagePath)\Unity.Entities.Analyzer.CodeFixes.dll" />
		<Analyzer Include="$(EntityPackagePath)\Unity.Entities.SourceGen.AspectGenerator.dll" />
		<Analyzer Include="$(EntityPackagePath)\Unity.Entities.SourceGen.Common.dll" />
		<Analyzer Include="$(EntityPackagePath)\Unity.Entities.SourceGen.SystemGenerator.EntityQueryBulkOperations.dll" />
		<Analyzer Include="$(EntityPackagePath)\Unity.Entities.SourceGen.JobEntityGenerator.dll" />
	</ItemGroup>
	<ItemGroup>
		<None Remove="Logs\**" />
		<None Remove="Library\**" />
	</ItemGroup>
	
</Project>
```

</details>

### `ModPostProcessor.Resources.ModSample.csproj`

<details>

```xml
<Project Sdk="Microsoft.NET.Sdk">

	<Import Project="..\Configuration.csproj" />
	<Import Project="..\Targets.csproj" />

	<ItemGroup>
		<Reference Include="Game">
			<Private>false</Private>
		</Reference>
		<Reference Include="Colossal.Core">
			<Private>false</Private>
		</Reference>
		<Reference Include="Colossal.Logging">
			<Private>false</Private>
		</Reference>
		<Reference Include="UnityEngine.CoreModule">
			<Private>false</Private>
		</Reference>
		<Reference Include="Unity.Burst">
			<Private>false</Private>
		</Reference>
		<Reference Include="Unity.Collections">
			<Private>false</Private>
		</Reference>
		<Reference Include="Unity.Entities">
			<Private>false</Private>
		</Reference>
		<Reference Include="Unity.Mathematics">
			<Private>false</Private>
		</Reference>
	</ItemGroup>

	<ItemGroup>
		<Reference Update="System">
			<Private>false</Private>
		</Reference>
	</ItemGroup>
	<ItemGroup>
		<Reference Update="System.Core">
			<Private>false</Private>
		</Reference>
	</ItemGroup>
	<ItemGroup>
		<Reference Update="System.Data">
			<Private>false</Private>
		</Reference>
	</ItemGroup>
	<ItemGroup>
		<Reference Update="System.Drawing">
			<Private>false</Private>
		</Reference>
	</ItemGroup>
	<ItemGroup>
		<Reference Update="System.IO.Compression.FileSystem">
			<Private>false</Private>
		</Reference>
	</ItemGroup>
	<ItemGroup>
		<Reference Update="System.Numerics">
			<Private>false</Private>
		</Reference>
	</ItemGroup>
	<ItemGroup>
		<Reference Update="System.Runtime.Serialization">
			<Private>false</Private>
		</Reference>
	</ItemGroup>
	<ItemGroup>
		<Reference Update="System.Xml">
			<Private>false</Private>
		</Reference>
	</ItemGroup>
	<ItemGroup>
		<Reference Update="System.Xml.Linq">
			<Private>false</Private>
		</Reference>
	</ItemGroup>

</Project>

```

</details>

### `ModPostProcessor.Resources.Targets.csproj`

<details>

```xml
<Project>
	
	<Target Name="ChechManagedDLLPath" BeforeTargets="PreBuildEvent">
		<Message Text="$(CommonLocation)\General.targets" Importance="high"/>
		<Error Condition="!Exists('$(ManagedDLLPath)')" Text="The Managed DLL path is wrong: $(ManagedDLLPath)" />
		<OnError ExecuteTargets="CheckInstallationPath" />
	</Target>
	
	<Target Name="CheckInstallationPath" AfterTargets="ChechManagedDLLPath">
		<Error Condition="'$(Configuration)'=='Steam' AND !Exists('$(InstallationPath)\Cities2.exe')" Text="The Game Installation path is wrong: $(InstallationPath)" />
		<OnError ExecuteTargets="CheckDataPath" />
	</Target>

	<Target Name="CheckDataPath" AfterTargets="CheckInstallationPath">
		<Error Condition="!Exists('$(DataPath)')" Text="The Game Data path is wrong: $(DataPath)" />
		<OnError ExecuteTargets="CheckUnityModProjectPath" />
	</Target>

	<Target Name="CheckUnityModProjectPath" AfterTargets="CheckDataPath">
		<Error Condition="!Exists('$(UnityModProjectPath)')" Text="The Unity Mod Project path is wrong: $(UnityModProjectPath)" />
		<OnError ExecuteTargets="CheckModPostProcessorPath" />
	</Target>

	<Target Name="CheckModPostProcessorPath" AfterTargets="CheckUnityModProjectPath">
		<Error Condition="!Exists('$(ModPostProcessorPath)')" Text="The Mod Post Processor path is wrong: $(ModPostProcessorPath)" />
		<OnError ExecuteTargets="CheckEntityPackagePath" />
	</Target>

	<Target Name="CheckEntityPackagePath" AfterTargets="CheckModPostProcessorPath">
		<Error Condition="!Exists('$(EntityPackagePath)')" Text="The Entity package path is wrong: $(EntityPackagePath)" />
	</Target>
	
	<Target Name="BeforeBuildAction" BeforeTargets="BeforeBuild">
		<RemoveDir Directories="$(OutDir)" />
	</Target>
	
	<Target Name="AfterBuildAction" AfterTargets="AfterBuild">
		<PropertyGroup>
			<ModPostProcessorArgs>"$(ModPostProcessorPath)" PostProcess "$(TargetPath)" -r "@(ReferencePath)" -u "$(UnityModProjectPath)" -t Windows -t macOS -t Linux -b $(Configuration) -d -v</ModPostProcessorArgs>
		</PropertyGroup>
		<Message Condition="Exists('$(ModPostProcessorPath)')" Text="Run post processor: $(ModPostProcessorArgs)" Importance="high" />
		<Message Condition="!Exists('$(ModPostProcessorPath)')" Text="Post processor was not found, please check the path: @(ModPostProcessorPath)" Importance="high" />
		<Exec Condition="Exists('$(ModPostProcessorPath)')" Command="$(ModPostProcessorArgs)"></Exec>
		<ItemGroup>
			<FilesToCopy Include="$(OutDir)\**\*.*" />
			<DeployDir Include="$(DataPath)\Mods\WorkInProgress\$(ProjectName)" />
		</ItemGroup>
		<Message Text="Copy output to deploy dir @(DeployDir)" Importance="high" />
		<RemoveDir Directories="@(DeployDir)" />
		<Copy SourceFiles="@(FilesToCopy)" DestinationFolder="@(DeployDir)" />
	</Target>
	
</Project>
```

</details>