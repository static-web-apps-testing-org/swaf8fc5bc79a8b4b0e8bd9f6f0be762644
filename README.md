# .NET 10 isolated API — Runner V1

This standalone fixture is the .NET 10 counterpart of `dotnet8-api`.
Runner uploads this entire directory as the source repository, with `src` as
the static app location and `api` as the Functions location. The app's
`src/staticwebapp.config.json` explicitly selects `dotnet-isolated:10.0`.
The Runner host itself remains on .NET 8.

## Cases

- `TestCases/dotnet10-api-create.json` — **DotNet 10 API Create** (`Sev2`):
  `/content.txt` returns HTTP 200 with `testcontent`, and `/api/GetMessage`
  returns HTTP 200 with `message content`.
- `TestCases/dotnet10-api-update-api.json` — **DotNet 10 API Update Api**
  (`Email`): validates the initial responses, then overlays the shared
  `TestAppCode/update-api` directory. The API response becomes
  `message content2`, while static content remains `testcontent`.

Both cases retain the .NET 8 cases' per-MSHA checks and deployment waits.
Response comparisons are exact; do not add newlines to either `content.txt`.
The anonymous `GetMessage` function accepts GET and POST and reads the API
content file on each invocation.

## Local build (no cloud resources)

Use a .NET 10 SDK. Build a copy **outside this repository**, matching Runner's
uploaded layout and avoiding the parent `Directory.Build.props` and central
package versions. Do not build inside `TestAppCode`: Runner uploads files
recursively and must not pick up local `bin` or `obj` output.

From the repository root on Linux/macOS:

```sh
validation_dir="$(mktemp -d)"
cp -R src/Runner/TestAppCode/dotnet10-api "$validation_dir/app"
dotnet publish "$validation_dir/app/api/dotnet-basic.csproj" \
  --configuration Release --output "$validation_dir/publish" \
  --source https://packagefeedproxy.microsoft.io/nuget/v3/index.json
```

Use your environment's approved NuGet feed when reproducing this outside
the engineering environment. Dependencies are pinned to Worker 2.52.0,
Worker SDK 2.0.7 and HTTP extension 3.1.0 using the existing
`Microsoft.NET.Sdk`/`ConfigureFunctionsWorkerDefaults` pattern.

Check that the publish directory contains:

- `dotnet-basic.runtimeconfig.json` targeting `net10.0`;
- `worker.config.json` identifying `dotnet-isolated`;
- `functions.metadata` declaring the anonymous GET/POST `GetMessage` HTTP
  trigger and HTTP return binding;
- `host.json` and `content.txt` with the exact initial API response.

To check update artifacts locally, overlay `TestAppCode/update-api/api/content.txt`
onto the copied API's `content.txt`, then republish. Only the response content
should change; the runtime, HTTP bindings and static app remain unchanged.
Local publishing does not validate routing, MSHA behavior or a live deployment.

## Private validation and enablement prerequisites

Deploy compatible .NET 10 Client/C2GP support
([PR 17242490](https://dev.azure.com/msazure/One/_git/AAPT-Antares-StaticWebsites/pullrequest/17242490))
and corresponding BlueRidge runtime support in the target private environment
before attempting Runner E2E. A successful local build is not evidence that
those external prerequisites have been deployed.

Runner V1 does **not** discover JSON cases automatically:
`TestExecutionHelper` reads the explicit `Constants.TestCases` list, where both
.NET 10 cases are registered beside their .NET 8 counterparts. Existing cases,
startup configuration and runtime catalogs remain unchanged.
Validate a private Runner build first. Do not deploy this Runner build to
production until runtime support and private E2E results are confirmed; the
registered create case uses the existing `Sev2` failure convention.
Starting Runner provisions and deletes cloud resources; it is not a local test.
