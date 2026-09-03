# elixir_phased_compiler_jusris_os
A Phased compiler for Elixir very very very especific case, I need review from especilits 

<img width="478" height="123" alt="image" src="https://github.com/user-attachments/assets/71dea006-d1a7-47d4-b9bd-edcee6812069" />

```elixir 
defmodule Mix.Tasks.Compile.Phased do
  use Mix.Task.Compiler

  @recursive true

  # Manifest "público" (lido por `mix xref`, `mix clean`, etc). É o resultado da
  # fusão dos dois manifests de fase abaixo, para manter a compatibilidade com as
  # ferramentas que leem o `compile.elixir` padrão.
  @manifest "compile.elixir"

  # Manifests privados, um por fase. Separados para que a fase 1 (core) não
  # trate os arquivos de plugin como "removidos" (e vice-versa) na compilação
  # incremental — problema inerente a usar um único manifest com dois `srcs`
  # diferentes.
  @core_manifest "compile.elixir.core"
  @proto_manifest "compile.elixir.proto"
  @plugins_manifest "compile.elixir.plugins"

  # Arquivos de protobuf GERADOS (determinísticos). São compilados de forma
  # incremental — NUNCA com `--force` — para reutilizar o `.beam` já existente
  # quando o fonte não mudou. São os arquivos mais caros do build (ex.:
  # `message.ex` tem ~100KB e define dezenas de módulos).
  @proto_glob "lib/**/proto/*.ex"

  # Raiz onde os plugins (arquitetura microkernel, shared-nothing) vivem.
  # Cada plugin é `lib/jusris_os_core/plugins/<slug>.ex` (entrypoint) +
  # `lib/jusris_os_core/plugins/<slug>/**` (LiveViews, schemas, etc).
  #
  # `plugins.ex` (agregador `JusrisOsCore.Plugins`) fica em
  # `lib/jusris_os_core/plugins.ex` (PAI da pasta `plugins/`) e `registry.ex`
  # (`JusrisOsCore.Plugins.Registry`) é infraestrutura do core — ambos são
  # compilados na fase 1 (core), NÃO na fase de plugins.
  @plugins_root "lib/jusris_os_core/plugins"

  @moduledoc """
  Compilador Elixir do JusrisOS, em duas fases, priorizando o núcleo do sistema.

  Substitui o compilador `compile.elixir` padrão no pipeline de `mix compile`
  (registrado como `:phased` em `mix.exs`). Usa um manifest por fase e funde o
  resultado em `compile.elixir` — preservando `mix xref`, `mix clean`,
  consolidação de protocolos, `__mix_recompile__?/0` e compilação incremental.

  ## Fase 1 — Core (prioritário)

  Compila primeiro todo o núcleo e suas dependências diretas — `JusrisOsCore`
  (`kernel`, `support`, `ports`, `adapters`, `types`, `validators`, `snowflake`,
  `sync`), a infraestrutura `JusrisOs` e o shell `JusrisOsWeb`. Garante que
  erros de compilação do núcleo apareçam **antes** de compilar os plugins, e que
  os plugins sempre enxerguem um core já compilado e consistente no code path.

  ## Fase 2 — Plugins (em paralelo)

  Depois compila os plugins (`lib/jusris_os_core/plugins/**`) de uma só vez, em
  paralelo, via o `Kernel.ParallelCompiler` (que distribui os arquivos entre
  todos os schedulers). Como os plugins são shared-nothing — só dependem de
  `Kernel`/`Support`, nunca uns dos outros — a compilação paralela é segura.

  Ao final, dispara os hooks `after_compiler(:elixir)` registrados pelos
  compiladores `:boundary` e `:phoenix_live_view` (descarregamento do tracer de
  boundary e compilação dos assets colocalizados do LiveView), preservando o
  comportamento do pipeline padrão.

  ## Opções

  Aceita as mesmas opções do compilador `compile.elixir` padrão (veja
  `mix help compile.elixir`): `--force`, `--warnings-as-errors`, `--verbose`,
  `--profile time`, `--no-protocol-consolidation`, etc.
  """

  @switches [
    force: :boolean,
    docs: :boolean,
    consolidate_protocols: :boolean,
    warnings_as_errors: :boolean,
    ignore_module_conflict: :boolean,
    debug_info: :boolean,
    verbose: :boolean,
    long_compilation_threshold: :integer,
    long_verification_threshold: :integer,
    purge_consolidation_path_if_stale: :string,
    purge_compiler_modules: :boolean,
    profile: :string,
    all_warnings: :boolean,
    verification: :boolean,
    tracer: :keep,
    check_cwd: :boolean
  ]

  @impl true
  def run(args) do
    {opts, _, _} = OptionParser.parse(args, switches: @switches)
    {tracers, opts} = pop_tracers(opts)

    project = Mix.Project.config()
    dest = Mix.Project.compile_path(project)
    srcs = project[:elixirc_paths]

    if not (is_list(srcs) and Enum.all?(srcs, &is_binary/1)) do
      Mix.raise(":elixirc_paths should be a list of string paths, got: #{inspect(srcs)}")
    end

    # Infer signatures should not trigger a full recompile but it does trigger rechecking
    {infer_signatures, base} =
      (project[:elixirc_options] || [])
      |> xref_exclude_opts(project)
      |> Keyword.pop(:infer_signatures, true)

    # Optional dependencies and debug info affect how artifacts are generated.
    # O cache key é computado sobre os `srcs` COMPLETOS e é o MESMO para as duas
    # fases (mantém consistência entre os manifests).
    cache_key = {base, srcs, "--no-optional-deps" in args, "--no-debug-info" in args}

    opts =
      base
      |> Keyword.merge(opts)
      |> tracers_opts(tracers)
      |> profile_opts()
      |> Keyword.put(:infer_signatures, infer_signatures(infer_signatures))

    opts =
      if "--no-protocol-consolidation" in args do
        Keyword.put(opts, :consolidate_protocols, false)
      else
        opts ++ Keyword.take(project, [:consolidate_protocols])
      end

    result =
      with_logger_app(project, fn ->
        Mix.Project.with_build_lock(project, fn ->
          compile_phased(srcs, dest, cache_key, opts)
        end)
      end)

    # Dispara os hooks que o pipeline padrão associa ao compilador `:elixir`
    # (registrados por `:boundary` e `:phoenix_live_view`): descarregamento do
    # tracer de boundary e compilação dos assets colocalizados do LiveView.
    Enum.reduce(Mix.ProjectStack.pop_after_compiler(:elixir), result, fn fun, acc ->
      fun.(acc)
    end)
  end

  @impl true
  def manifests, do: [manifest(), core_manifest(), proto_manifest(), plugins_manifest()]
  defp manifest, do: Path.join(Mix.Project.manifest_path(), @manifest)
  defp core_manifest, do: Path.join(Mix.Project.manifest_path(), @core_manifest)
  defp proto_manifest, do: Path.join(Mix.Project.manifest_path(), @proto_manifest)
  defp plugins_manifest, do: Path.join(Mix.Project.manifest_path(), @plugins_manifest)

  # Módulos compilados na fase de protobuf, lidos do manifest do proto. Usados
  # como "parents" da fase de plugins para propagar mudanças de API do proto.
  defp proto_modules do
    case Mix.Compilers.Elixir.read_manifest(proto_manifest()) do
      {modules, _sources} -> Map.keys(modules)
    end
  end

  @impl true
  def diagnostics do
    Mix.Compilers.Elixir.diagnostics(core_manifest()) ++
      Mix.Compilers.Elixir.diagnostics(proto_manifest()) ++
      Mix.Compilers.Elixir.diagnostics(plugins_manifest())
  end

  @impl true
  def clean do
    dest = Mix.Project.compile_path()

    # O manifest fundido contém todos os módulos; limpá-lo remove todos os
    # beams. Os manifests de fase são então removidos individualmente.
    Mix.Compilers.Elixir.clean(manifest(), dest)
    File.rm(core_manifest())
    File.rm(proto_manifest())
    File.rm(plugins_manifest())
    :ok
  end

  # ---------------------------------------------------------------------------
  # Compilação em fases
  # ---------------------------------------------------------------------------

  defp compile_phased(srcs, dest, cache_key, opts) do
    erlang_manifests = Mix.Tasks.Compile.Erlang.manifests()
    erlang_modules = Mix.Tasks.Compile.Erlang.modules()

    {core_srcs, proto_srcs, plugin_srcs} = partition_sources(srcs)

    # Na fase 1 o core referencia módulos de plugins em RUNTIME (ex.: o
    # `MasterState` registra os schemas de todos os plugins), que ainda não
    # foram compilados — o que geraria falsos avisos "module is not available".
    # Por isso, nesta fase:
    #
    #   * `no_warn_undefined: :all` — suprime os avisos de módulo indefinido
    #     (as referências são legítimas e serão resolvidas nas fases seguintes);
    #   * `infer_signatures: false` — além de desligar a inferência, muda o
    #     `infer_signatures` gravado no manifest do core; isso dispara o
    #     `reinfer?` na fase de re-checagem, que RE-VERIFICA os módulos do core
    #     com os plugins já compilados — então módulos GENUINAMENTE inexistentes
    #     continuam sendo reportados (na re-checagem, com contexto correto);
    #   * `consolidate_protocols: false` — adia a consolidação de protocolos
    #     para a fase de plugins, quando as `defimpl` já foram compiladas.
    core_opts =
      opts
      |> Keyword.put(:consolidate_protocols, false)
      |> Keyword.put(:infer_signatures, false)
      |> Keyword.put(:no_warn_undefined, :all)

    announce_phase(1, "core + shell", core_srcs)

    case Mix.Compilers.Elixir.compile(
           core_manifest(),
           core_srcs,
           dest,
           cache_key,
           erlang_manifests,
           erlang_modules,
           core_opts
         ) do
      {:error, _diagnostics} = error ->
        # Erro no core: aborta sem compilar o restante.
        error

      {core_status, core_diagnostics} ->
        # Fase 2: protobuf gerado (determinístico), compilado SEMPRE de forma
        # incremental — mesmo sob `--force` — para reaproveitar o `.beam`
        # existente quando o fonte não mudou.
        announce_phase(2, "proto (cacheado)", proto_srcs)

        proto_opts =
          opts
          |> Keyword.delete(:force)
          |> Keyword.put(:consolidate_protocols, false)

        case Mix.Compilers.Elixir.compile(
               proto_manifest(),
               proto_srcs,
               dest,
               cache_key,
               erlang_manifests,
               erlang_modules,
               proto_opts
             ) do
          {:error, _diagnostics} = error ->
            error

          {proto_status, proto_diagnostics} ->
            announce_phase(3, "plugins (paralelo)", plugin_srcs)

            # Os plugins dependem dos módulos de protobuf (manifest próprio).
            # Passamos o manifest do proto como "parent": quando ele muda (proto
            # recompilado), os plugins que referenciam seus módulos são
            # recompilados — preservando a corretude sem abrir mão do cache.
            plugins_parent_manifests = erlang_manifests ++ [proto_manifest()]
            plugins_parent_modules = erlang_modules ++ proto_modules()

            case Mix.Compilers.Elixir.compile(
                   plugins_manifest(),
                   plugin_srcs,
                   dest,
                   cache_key,
                   plugins_parent_manifests,
                   plugins_parent_modules,
                   opts
                 ) do
              {:error, _diagnostics} = error ->
                error

              {plugin_status, plugin_diagnostics} ->
                # Re-checagem do core (somente quando o core foi de fato
                # compilado na fase 1). Como os plugins já foram compilados e
                # estão no code path, os módulos de plugin agora resolvem; apenas
                # referências GENUINAMENTE inexistentes geram aviso (com o
                # `no_warn_undefined` padrão, não `:all`). O `reinfer?`
                # (disparado pela troca de `infer_signatures`) faz o compilador
                # re-verificar sem recompilar.
                if core_status == :ok do
                  # Sem `:force` aqui: a re-checagem só RE-VERIFICA (via
                  # `reinfer?`), não recompila. O core já foi compilado na fase 1.
                  recheck_opts =
                    opts
                    |> Keyword.put(:consolidate_protocols, false)
                    |> Keyword.delete(:force)

                  case Mix.Compilers.Elixir.compile(
                         core_manifest(),
                         core_srcs,
                         dest,
                         cache_key,
                         erlang_manifests,
                         erlang_modules,
                         recheck_opts
                       ) do
                    {:error, _diagnostics} = error ->
                      error

                    {recheck_status, recheck_diagnostics} ->
                      finalize(
                        core_status,
                        proto_status,
                        plugin_status,
                        recheck_status,
                        core_diagnostics,
                        proto_diagnostics,
                        plugin_diagnostics,
                        recheck_diagnostics
                      )
                  end
                else
                  finalize(
                    core_status,
                    proto_status,
                    plugin_status,
                    :noop,
                    core_diagnostics,
                    proto_diagnostics,
                    plugin_diagnostics,
                    []
                  )
                end
            end
        end
    end
  end

  # Funde os manifests de fase no `compile.elixir` canônico e consolida o status
  # final. Chamado quando todas as fases terminam com sucesso/noop.
  defp finalize(
         core_status,
         proto_status,
         plugin_status,
         recheck_status,
         core_diags,
         proto_diags,
         plugin_diags,
         recheck_diags
       ) do
    merge_manifests([core_manifest(), proto_manifest(), plugins_manifest()], manifest())

    status =
      if core_status == :ok or proto_status == :ok or plugin_status == :ok or
           recheck_status == :ok,
         do: :ok,
         else: :noop

    {status, core_diags ++ proto_diags ++ plugin_diags ++ recheck_diags}
  end

  defp announce_phase(_phase, _label, []) do
    :ok
  end

  defp announce_phase(phase, label, srcs) do
    Mix.shell().info("==> Fase #{phase} — compilando #{label} (#{length(srcs)} arquivo(s))")
  end

  # Divide os fontes em `{core, proto, plugins}`.
  #
  # * **proto**: arquivos de protobuf gerado (determinísticos), detectados pelo
  #   glob `@proto_glob`. Compilados em manifest próprio, SEMPRE de forma
  #   incremental — reaproveitam o `.beam` existente quando o fonte não mudou.
  # * **plugins**: o restante dos plugins.
  # * **core**: tudo o mais.
  #
  # Um arquivo é "de plugin" quando pertence a um plugin registrado, por
  # DIRETÓRIO (entrypoint `lib/jusris_os_core/plugins/<slug>.ex` ou subdir
  # `lib/jusris_os_core/plugins/<slug>/**`) OU por NAMESPACE de módulo (define
  # `JusrisOsCore.Plugins.<slug>` / `JusrisOsCore.Plugins.<slug>.*`). O segundo
  # cobre arquivos de suporte colocalizados FORA do diretório do plugin — ex.:
  # `test/support/mock_wa_server.ex` define
  # `JusrisOsCore.Plugins.Baileys.TestSupport.MockWAServer` e, por isso, deve ser
  # compilado na fase de plugins (depende de structs do baileys em tempo de
  # compilação). O slug é descoberto pelos subdiretórios de `plugins/` (fonte de
  # verdade no filesystem, sem depender de módulos já compilados).
  defp partition_sources(srcs) do
    all_files = Mix.Utils.extract_files(srcs, [:ex]) |> Enum.sort()
    slugs = plugin_slugs()
    proto_set = @proto_glob |> Path.wildcard() |> MapSet.new()

    proto_files = Enum.filter(all_files, &MapSet.member?(proto_set, &1))

    plugin_files =
      Enum.filter(all_files, fn file ->
        not MapSet.member?(proto_set, file) and plugin_file?(file, slugs)
      end)

    core_files = all_files -- (proto_files ++ plugin_files)
    {core_files, proto_files, plugin_files}
  end

  defp plugin_slugs do
    @plugins_root
    |> Path.join("*")
    |> Path.wildcard()
    |> Enum.filter(&File.dir?/1)
    |> Enum.map(&Path.basename/1)
    |> Enum.sort()
  end

  defp plugin_file?(file, slugs) do
    directory_match?(file, slugs) or namespace_match?(file, slugs)
  end

  defp directory_match?(file, slugs) do
    Enum.any?(slugs, fn slug ->
      prefix = @plugins_root <> "/" <> slug
      file == prefix <> ".ex" or String.starts_with?(file, prefix <> "/")
    end)
  end

  defp namespace_match?(file, slugs) do
    Enum.any?(defmodules(file), fn module ->
      Enum.any?(slugs, fn slug ->
        # O diretório do plugin é minúsculo (`plugins/baileys`), mas o namespace
        # do módulo é CamelCase (`JusrisOsCore.Plugins.Baileys`).
        name = Macro.camelize(slug)

        module == "JusrisOsCore.Plugins." <> name or
          String.starts_with?(module, "JusrisOsCore.Plugins." <> name <> ".")
      end)
    end)
  end

  # Extrai os nomes de `defmodule` de um arquivo (nomes controlados, dos nossos
  # próprios fontes — nunca input de usuário).
  defp defmodules(file) do
    file
    |> File.read!()
    |> then(&Regex.scan(~r/defmodule\s+([A-Z][A-Za-z0-9_]*(?:\.[A-Z][A-Za-z0-9_]*)*)/, &1))
    |> Enum.map(fn [_full, name] -> name end)
  end

  # ---------------------------------------------------------------------------
  # Fusão dos manifests de fase no manifest canônico (`compile.elixir`)
  # ---------------------------------------------------------------------------

  # O formato do manifest é o mesmo escrito por `Mix.Compilers.Elixir`:
  #
  #   {vsn, modules, sources, exports, parents, cache_key, cwd, deps_config,
  #    project_mtime, config_mtime, protocols_and_impls}
  #
  # Fundimos os mapas `modules`, `sources`, `exports` e `protocols_and_impls` de
  # todos os manifests de fase; o restante (metadata) vem do primeiro manifest
  # válido. O `vsn` é lido do próprio manifest (escrito pela versão atual do
  # Elixir), sem hardcode.
  defp merge_manifests(source_manifests, dest_manifest) do
    case Enum.map(source_manifests, &read_raw_manifest/1) do
      [{vsn, _, _, _, _, _, _, _, _, _, _} = base | rest] when is_integer(vsn) ->
        if Enum.all?(rest, &match?({^vsn, _, _, _, _, _, _, _, _, _, _}, &1)) do
          merged = Enum.reduce(rest, base, &merge_raw/2)

          File.mkdir_p!(Path.dirname(dest_manifest))
          File.write!(dest_manifest, :erlang.term_to_binary(merged, [:compressed]))
        end

        :ok

      _ ->
        :ok
    end
  end

  # Funde dois manifests crus de mesma `vsn` (verificado pelo chamador).
  defp merge_raw(
         {vsn, m1, s1, e1, parents, cache_key, cwd, deps_config, project_mtime, config_mtime, p1},
         {vsn, m2, s2, e2, _, _, _, _, _, _, p2}
       ) do
    {
      vsn,
      Map.merge(m1, m2),
      Map.merge(s1, s2),
      Map.merge(e1, e2),
      parents,
      cache_key,
      cwd,
      deps_config,
      project_mtime,
      config_mtime,
      merge_protocols(p1, p2)
    }
  end

  defp read_raw_manifest(manifest) do
    manifest
    |> File.read!()
    |> :erlang.binary_to_term()
  rescue
    _ -> nil
  end

  defp merge_protocols({core_protocols, core_impls}, {plugin_protocols, plugin_impls}) do
    {Map.merge(core_protocols, plugin_protocols), Map.merge(core_impls, plugin_impls)}
  end

  # ---------------------------------------------------------------------------
  # Helpers (espelham o compilador `compile.elixir` padrão do Elixir)
  # ---------------------------------------------------------------------------

  defp infer_signatures(list) when is_list(list), do: :lists.usort([:elixir | list])

  defp infer_signatures(false) do
    false
  end

  defp infer_signatures(true) do
    project = Mix.Project.get!()

    properties =
      if function_exported?(project, :application, 0), do: project.application(), else: []

    extra_apps =
      for tuple_or_atom <-
            Keyword.get(properties, :included_applications, []) ++
              Keyword.get(properties, :extra_applications, []) do
        case tuple_or_atom do
          {app, _} -> app
          app when is_atom(app) -> app
        end
      end

    deps_apps =
      for %{app: app, opts: opts} <- Mix.Dep.cached(),
          Keyword.get(opts, :app, true),
          do: app

    :lists.usort([:elixir | extra_apps] ++ deps_apps)
  end

  # Run this operation in compile.elixir as the compiler can be called directly
  defp with_logger_app(config, fun) do
    app = Keyword.fetch!(config, :app)
    logger_config_app = Application.get_env(:logger, :compile_time_application)

    try do
      Application.put_env(:logger, :compile_time_application, app)
      fun.()
    after
      Application.put_env(:logger, :compile_time_application, logger_config_app)
    end
  end

  defp xref_exclude_opts(opts, project) do
    exclude = List.wrap(project[:xref][:exclude])

    if exclude == [] do
      opts
    else
      IO.warn(
        "\"xref: [exclude: ...]\" in your mix.exs file is deprecated, " <>
          "instead use: \"elixirc_options: [no_warn_undefined: ...]\""
      )

      Keyword.update(opts, :no_warn_undefined, exclude, &(List.wrap(&1) ++ exclude))
    end
  end

  defp pop_tracers(opts) do
    case Keyword.pop_values(opts, :tracer) do
      {[], opts} ->
        {[], opts}

      {tracers, opts} ->
        {Enum.map(tracers, &Module.concat([&1])), opts}
    end
  end

  defp tracers_opts(opts, tracers) do
    tracers = tracers ++ Code.get_compiler_option(:tracers)
    Keyword.update(opts, :tracers, tracers, &(tracers ++ &1))
  end

  defp profile_opts(opts) do
    case Keyword.fetch(opts, :profile) do
      {:ok, "time"} -> Keyword.put(opts, :profile, :time)
      {:ok, _} -> Keyword.delete(opts, :profile)
      :error -> opts
    end
  end
end


```
