FROM microsoft/aspnetcore-build AS build-env
WORKDIR /app

COPY *.csproj ./
RUN dotnet restore

COPY . ./
RUN dotnet publish -c Release -o output

FROM microsoft/aspnetcore
WORKDIR /app
COPY --from=build-env /app/output .
ENTRYPOINT ["dotnet", "AksWorkshop.dll"]