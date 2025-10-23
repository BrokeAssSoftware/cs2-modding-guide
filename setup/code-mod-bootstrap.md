# Bootstrap a Code Mod

Generate the base C# project and move it into the Vice & Order repository so you can begin extending the simulation.

1. Open *Options -> Modding -> Projects* and click *Create*.
2. Select the **Code Mod** template and enter the module ID (for example `VNO.Core` or `VNO.Vice`).
3. The toolchain writes a solution to `%LocalAppData%\Colossal Order\Cities Skylines II\Mods\Local\<ModuleId>`.
4. Move the generated project into the repository (for example `vno-core/`) and add it to source control.
5. Populate `Properties/PublishConfiguration.xml` with placeholder metadata so publish scripts have the required fields.
6. Open the solution in Rider or Visual Studio, restore packages, and run `dotnet build` once to verify the pipeline.

Next: [Bootstrap a UI Project](ui-project-bootstrap.md) if the module ships Gameface components.
