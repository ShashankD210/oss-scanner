# The scanner supplies the Fluence.Wpf checkout as the build context. Its audit
# runs with no network, so restore every solution project's locked dependency
# graph while the image is being prepared.
FROM mcr.microsoft.com/dotnet/sdk:10.0

COPY . /src
WORKDIR /src

# The repository enables Windows targeting in Directory.Build.props. Restoring
# the original target frameworks preserves the committed NuGet lock files.
RUN dotnet restore Fluence.Wpf.sln --locked-mode

# Debug builds retain portable PDBs. These Windows-targeted binaries can be
# inspected in this Linux image; running WPF applications and tests needs Windows.
RUN dotnet build Fluence.Wpf/Fluence.Wpf.csproj \
      -c Debug -f net10.0-windows10.0.26100.0 --no-restore \
    && dotnet build Fluence.Wpf/Fluence.Wpf.csproj \
      -c Debug -f net8.0-windows10.0.26100.0 --no-restore

# Offline rebuilds inside the finished scanner image can use the cached graph:
#   dotnet build Fluence.Wpf/Fluence.Wpf.csproj -c Debug \
#     -f net10.0-windows10.0.26100.0 --no-restore
# The compiled DLL/PDB files stay under each project's bin/Debug/<TFM>/.
