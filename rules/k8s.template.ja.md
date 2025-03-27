# Kubernetes スタイルガイドおよびベストプラクティスガイド

**注意：** このガイドでは、**RFC 2119** および **RFC 8174** のキーワード（MUST、MUST NOT、SHOULD、SHOULD NOT、MAY）を使用して各ルールの遵守レベルを示しています。これはプロダクションコードを書く中堅〜上級のKubernetesエンジニアを対象とし、大規模コードベースにおける一貫性、明確性、保守性を重視しています。

---

## 環境と Namespace の分離

- **[MUST]** 環境（開発、ステージング、本番）を明確に分離すること。クリティカルなワークロードを隔離するため、本番用には別々のクラスターを使用することを推奨する。コスト削減や管理の簡素化のために複数環境が同一クラスターを共有する場合は、必ず強固な分離措置を講じた異なる Namespace で運用しなければならない。（[Multi-tenancy | Kubernetes](https://kubernetes.io/docs/concepts/security/multi-tenancy/)）

  開発・テストのワークロードを本番 Namespace やクラスターに展開してはならない。

- **[MUST]** テナントおよびチームの分離を徹底すること。マルチテナントクラスターでは、各チームまたはプロジェクトが独自の Namespace を持つ。各チームのプロセスが、定義された API を通じた場合を除き、他チームに影響を及ぼしてはならない。これにはリソースクォータや Namespace 間の NetworkPolicy の適用が含まれる。（[Kubernetes Network Policies: Best Practices - Daily.dev](https://daily.dev/blog/kubernetes-network-policies-best-practices)）

- **[MUST]** 環境および所有者のラベル付けを行うこと。各 Namespace には、その環境（例：`env: prod` や `env: staging`）および所有チームやビジネスユニットを示すラベルを付けるべきだ。（[Securing Kubernetes: Using Gatekeeper to enforce effective security policies | SecureFlag](https://blog.secureflag.com/2024/03/13/security-policy-enforcement-in-kubernetes/)）

- **[SHOULD]** 一貫したクラスター命名規則を使用すること（例：`-prod` や `-dev` を含める）。これにより、事故を防げる。

- **[MUST]** 環境ごとに別々の設定を維持すること。設定差異（レプリカ数、エンドポイント、認証情報）は、Helm values ファイルや Kustomize オーバーレイで管理する。

- **[SHOULD]** 本番前にステージングでテストを行うこと。本番と同一環境で検証を行う。

---

## 命名およびメタデータの規約

- **[MUST]** リソース名は小文字で、ハイフン（`-`）を使用する。名前は目的を明確に記述すること。

- **[MUST]** リソース名にはアプリケーション名やサービス名を含めること。環境識別子を必要に応じて名前に含める。

- **[MUST]** ラベル：すべての Kubernetes オブジェクトには、少なくとも推奨ラベルを付与すること。

  ```yaml
  labels:
    app.kubernetes.io/name: payment-service
    app.kubernetes.io/instance: payment-service-abc123
    app.kubernetes.io/component: api
    app.kubernetes.io/version: "1.4.2"
  ```

- **[MUST]** ラベルセレクターとラベルが一致することを保証すること。

- **[SHOULD]** 識別目的以外のメタデータにはアノテーションを使用すること。

- **[MUST NOT]** 大きなデータや機密情報をラベルに含めてはならない。

- **[SHOULD]** チーム間で一貫した命名規約やラベル付け規約を使用すること。

---

## ワークロード（Deployment と Pod）

- **[MUST]** イメージのバージョン固定：すべてのコンテナイメージは特定のバージョンタグまたはダイジェストで指定すること。

- **[MUST]** 各コンテナにリソース要求および制限を必ず指定すること。

  ```yaml
  resources:
    requests:
      cpu: "200m"
      memory: "256Mi"
    limits:
      cpu: "500m"
      memory: "512Mi"
  ```

- **[MUST]** 各 Deployment には readinessProbe と livenessProbe を含めること。

- **[SHOULD]** プローブの設定は注意深く調整し、必要なら startupProbe を使用する。

- **[MUST]** 特権 Pod はデフォルトで使用しないこと。非 root ユーザーで実行する。

- **[MUST]** Pod は必要がない限り hostNetwork、hostPID、hostIPC、hostPort を使用しないこと。

- **[MUST]** 信頼できるコンテナイメージのみを使用すること。CI でイメージスキャンを実施する。

- **[MUST]** クリティカルなサービスの Deployment は複数レプリカで運用すること。

- **[SHOULD]** Pod は障害ドメインに分散させること。

  ```yaml
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            topologyKey: kubernetes.io/hostname
            labelSelector:
              matchLabels:
                app: payment-service
  ```

- **[SHOULD]** Deployment のアップデート戦略を適切に設定すること。

- **[SHOULD]** Pod Disruption Budget（PDB）を定義し、最低限の可用性を保証する。

- **[MUST NOT]** Kubernetes の仕組みを回避するアンチパターンを使用してはならない。

---

## サービスと Ingress

- [MUST] 内部通信には **ClusterIP** サービスを優先すること。デフォルトでは、タイプを指定せずにサービスを作成し、クラスター内部でアプリケーションを公開する。サービスディスカバリーには DNS（サービス名）を使用する。**内部のマイクロサービス間通信においては、NodePort や LoadBalancer は使用してはならない**。これらは外部アクセス用である。

- [MUST] **Service セレクター**: 各 Service には、対象となる Pod のラベルと一致する `spec.selector` を定義すること。これは明白だが重要である。セレクターが設定されていない（または誤った）Service は Pod を対象としないため、必ず Deployment の Pod テンプレートのラベルと完全一致させること。

- [SHOULD] 意味のある **サービス命名** を使用すること。例えば、サービスオブジェクトを区別するためにサフィックスとして `-svc` を付けたり、曖昧でなければアプリ名と同一にするなど、一貫性が重要である。アプリケーションに複数のサービスがある場合（例：HTTP ポートと gRPC ポート）、それぞれを明確に命名すること（例：`myapp-http` および `myapp-grpc`）。

- [MUST NOT] NodePort サービスを直接インターネット上に公開してはならない。NodePort を使用する場合（例：外部ロードバランサーのターゲットや開発環境）、外部ファイアウォールルールで保護するか、内部 IP のみに限定する必要がある。一般に、NodePort や type=LoadBalancer はクラウドプロバイダー固有の方法で外部トラフィックを取り込むものであるため、本番環境では Ingress やクラウドゲートウェイを使用してルーティングを制御すること。

- [MUST] 外部トラフィックには **TLS によるセキュリティ** を必ず適用すること。すべての Ingress オブジェクトまたは外部サービスは、エンドユーザーのトラフィックに対して TLS 暗号化を使用しなければならない。Ingress では、`spec.tls` 設定に有効な証明書（cert-manager やクラウド連携で管理）を使用すること。クラウドの LoadBalancer では、LB もしくは Pod で TLS を終了させ、証明書が適切に配置されるようにする。HTTP（ポート 80）は、HTTPS へのリダイレクトのみ許可してよい。

- [SHOULD] 多数の LoadBalancer サービスを作成するのではなく、外部 HTTP(S) トラフィックのルーティングを統合するために Kubernetes の **Ingress** リソース（または API Gateway 機構）を使用すること。Ingress コントローラー（Nginx、GKE Ingress、AWS ALB Ingress など）を使用することで、複数のサービスに対して 1 つまたは少数のロードバランサーフロントエンドを管理でき、コストや複雑さが軽減される。

- [MUST] Ingress ルールを定義する際、各アプリケーションまたはサービスに対して一意のホスト名を指定すること。**Ingress 内で同一のホスト/パスの組み合わせを複数のサービスが主張してはならない**。これにより未定義の動作が発生する。Ingress のホスト名は、必ず自分が管理する DNS 名である必要があり、グローバルに一意である（DNS 上で重複しない）ことが望ましい（[SecureFlag](https://blog.secureflag.com/2024/03/13/security-policy-enforcement-in-kubernetes/)）。

- [MUST] 同一 Namespace またはクラスター内で複数の Ingress オブジェクトを使用する場合、`IngressClass` を定義するか、異なるホスト名を使用して各 Ingress リソースが意図したコントローラーにより処理されるようにすること。例えば、ALB Ingress コントローラーを使用する EKS では、適切な Ingress クラスのアノテーションを使用し、GKE では必要に応じて `kubernetes.io/ingress.class: "gce"` を使用する。これにより、複数の Ingress コントローラーが存在する場合の誤ルーティングを防げる。

- [SHOULD] Ingress ルールは、シンプルかつ明示的に保つこと。ワイルドカードホストやキャッチオールパスは、必要不可欠でない限り避け、既知のホストやパスを列挙して明瞭かつセキュアにルーティングを行う。

- [SHOULD] 特殊な Ingress アノテーション（クラウド固有のものなど）がある場合は、必ず文書化すること。例えば、GKE Ingress では `kubernetes.io/ingress.global-static-ip-name`（静的 IP 用）や、AWS ALB Ingress では `alb.ingress.kubernetes.io/scheme`（インターネット向け vs 内部向け）などのアノテーションが使用される場合があり、これらは環境ごとの値（Helm やオーバーレイ）で設定できるようにすること。

- [MUST] サービスに対して **NetworkPolicy** を実装すること。特に、マルチテナントクラスターやセキュリティ上重要なアプリケーションの場合、Kubernetes はデフォルトで Pod 間の通信を全て許可するため、必要な通信のみを許可する NetworkPolicy を作成すべきである（[Daily.dev](https://daily.dev/blog/kubernetes-network-policies-best-practices)）。まず "deny all"（全拒否）を基本とし、必要なラベル間のみでポートを開放する。

- [MUST] サービスディスカバリーと接続性を検証すること。Service とそれに対応する Pod をデプロイした後、他の Pod から `ping myservice.namespace.svc.cluster.local` などで DNS 解決ができ、トラフィックが期待通りに流れているか確認すること。セレクターや NetworkPolicy の問題を早期に発見するため、これらのチェックをデプロイ検証プロセスに組み込む。

- [SHOULD] 通常のサービスルーティングをバイパスする必要があるクラスター内部の通信（例：安定したネットワーク ID を持つ StatefulSet や、データベースディスカバリー用のヘッドレスサービス）の場合、なぜ headless service（`clusterIP: None`）または Pod への直接アドレス指定を使用するのか、その理由と方法を文書化すること。これらは高度なユースケースであり、デフォルトは ClusterIP サービスを使用し、Pod IP に依存しないことが望ましい。

---

## セキュリティと分離

- [MUST] **Pod Security Standards (PSS)** をクラスター全体で遵守すること。Kubernetes 1.25 以降、PodSecurityPolicies (PSP) は廃止されているため、組み込みの Pod Security Admission コントローラーまたは同等の仕組みを使用して、すべての Namespace に対して **Baseline** ポリシーを適用し、可能な限り本番環境では **Restricted** ポリシーを適用しなければならない。これにより、Pod が特権モードで実行されたり、ホスト Namespace を使用するなどの動作がデフォルトで防止される。ワークロードは（例外がある場合を除き）これらの基準に準拠して構築される必要がある（root ユーザーの不使用、危険なマウントの回避など）。

- [MUST] **特権昇格の禁止**: すべてのアプリケーション Pod は、`securityContext.allowPrivilegeEscalation: false` で実行されなければならない（追加の機能がない場合はデフォルトでそうなる）。これにより、仮にアプリケーションが侵害されても、容易にノード上で root 権限を取得できなくなる。本番クラスターでは、Restricted PSS によりこのルール違反が検出される。

- [MUST] カスタムポリシーについては、**OPA/Gatekeeper** を使用して実装すること。Pod Security Standards でカバーされないセキュリティルールや規約については、Open Policy Agent (OPA) Gatekeeper を用いた policy-as-code により、すべての Pod にリソース制限、Readiness プローブの設定、許可されたレジストリからのイメージ使用、特定ラベル（例：`owner`）の存在などを検証できる柔軟な仕組みを構築すること。これにより、`kubectl apply` のたびにポリシーの検証が可能となる。

- [MUST] **ネットワークの分離**: 前述のサービス設定と同様に NetworkPolicy を適用して通信を制限すること。Namespace 内ではデフォルトですべての Pod 間通信が許可されているため、必要に応じてクロス Namespace の通信を制限する。各 Namespace でのインバウンドトラフィックに対してデフォルトで拒否ポリシーを実施し（例：共通の Namespace（Ingress コントローラーや DNS）を除く）、必要な場合にのみ特定 Namespace 間の通信を許可する。このゼロトラストのネットワーク姿勢により、侵害された Pod が他の Namespace に自由にアクセスすることを防げる。

- [MUST] **RBAC の最小権限原則** を適用すること。すべてのクラスターで Kubernetes の Role-Based Access Control (RBAC) を利用し、最小権限の原則に従う。具体的には:

  - アプリのデプロイを行う開発者や CI のサービスアカウントは、特定 Namespace 内のリソース作成/更新などの限定的な権限のみを持つ。
  - Pod 内で実行されるアプリケーションのサービスアカウントは、通常 Kubernetes API を呼び出さないため RBAC ロールは不要である。必要な場合のみ最小限の Role を定義しバインドする。
  - `cluster-admin` などの組み込みロールは、クラスター全体の管理操作以外では使用せず、Namespace 内で Pod を実行する際は専用のサービスアカウントを作成すること。
  - 定期的に RBAC バインディングを監査し、不要な権限は削除または厳格化すること。

- [MUST] **クラウド IAM との連携** を実施すること。クラウド環境（EKS、GKE など）で実行する場合、静的な認証情報ではなくクラウド IAM を活用し、AWS EKS では IAM Roles for Service Accounts (IRSA)、GKE では Workload Identity を使用してサービスアカウントと連携させる。これにより、クラウド API キーを Secret に保存する必要がなくなる。

- [MUST] **Kubernetes API へのアクセスを保護**すること。Kubernetes API サーバーへのアクセスを厳格に制御し、企業の SSO と連携した認証（OIDC など）を用いて人間のアクセスを管理すること。CI システムには限定された権限を使用させ、有効期限の短いトークンや期限付きの証明書を使用すること。

- [SHOULD] API サーバーで **監査ログ** を有効にし、分析すること。これにより、誰が何を行ったかを追跡し、不審な操作を検出できるようになる。

- [MUST] **イメージのセキュリティスキャン**: すべてのコンテナイメージは、Trivy、Aqua、Clair などのツールで定期的に脆弱性スキャンを実施すること。高リスクまたはクリティカルな脆弱性が検出された場合、本番環境へのデプロイ前に修正する。また、承認されたレジストリからのイメージのみを使用する。

- [SHOULD] **定期的なアップデートとパッチ適用**: クラスターのバージョンや依存関係を最新に保つこと。ワークロードのベースイメージも頻繁に更新し、脆弱性を防ぐためにセキュリティパッチを適用する。

- [SHOULD] **ペネトレーションテストおよび脅威モデリング** を定期的に実施し、Kubernetes 環境のセキュリティレビューを行うこと。Namespace 間の分離やネットワークポリシー、RBAC の有効性を検証する。

- [SHOULD] **ノードおよびホスト境界の保護** を徹底すること。ノードのハードニング（CIS ベンチマークの適用、不要パッケージの削除、Docker ではなく containerd 使用など）を実施する。GKE の COS やマネージド OS の使用を検討する。

- [MUST] **インシデント対応の準備** を行うこと。侵害が発生した場合に迅速に資格情報を無効化し、ワークロードを隔離・停止できる体制を整える。設定はソース管理下に置き、クリーンな状態で再デプロイ可能にする。また、異常な活動を監視する仕組みを導入する。

---

## Helm チャートのベストプラクティス

- [MUST] **チャートメタデータ**: 各 Helm チャートは、`Chart.yaml` に `name`、`version`、`appVersion` および説明フィールドを正しく記載しなければならない。`version` はセマンティックバージョニングを採用し、`appVersion` はデプロイするアプリケーションのバージョンを示す。マニフェストやデフォルト値の変更時にはチャートバージョンを更新し、リリース済みのバージョンは再利用してはならない。

- [MUST] **Values とデフォルト値**: `values.yaml` に適切なデフォルト値を設定すること。開発者がわずかな上書きで稼働可能になるようにし、重要な項目は必須とするか安全なデフォルト値を設定し、各項目の意味をコメントで明記する。

- [SHOULD] **Values の構造**: 論理的で簡潔な構造で値を整理し、関連する設定をグループ化すること。例えば：

  ```yaml
  replicaCount: 3
  image:
    repository: nginx
    tag: 1.21.6
    pullPolicy: IfNotPresent
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 200m
      memory: 256Mi
  service:
    type: ClusterIP
    port: 80
  ```

  ネストが深すぎず、トップレベルキーが多すぎないように注意する。

- [SHOULD] **テンプレートの命名規則**: Helm テンプレートは有効な Kubernetes YAML を生成すること。`helm lint` で一般的なミスを検出し、インデントは `{{- ... -}}` 構文と `nindent` を使って適切に制御する。条件付きブロック例：

  ```yaml
  {{- if .Values.serviceAccount.create }}
  serviceAccountName: {{ include "<chartname>.fullname" . }}-sa
  {{- end }}
  ```

- [SHOULD] **再利用のためのヘルパーの利用**: 共通のテンプレートコードを `_helpers.tpl` に定義し再利用すること。例：

  ```yaml
  {{- define "<chartname>.labels" -}}
  app.kubernetes.io/name: {{ include "<chartname>.name" . }}
  helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version | replace "+" "_" }}
  app.kubernetes.io/instance: {{ .Release.Name }}
  app.kubernetes.io/managed-by: {{ .Release.Service }}
  {{- end -}}
  ```

- [MUST] **環境固有のハードコードは避ける**: チャートのテンプレートはクラウド非依存であること。環境固有のアノテーションは values ファイル経由で設定できるように設計すること。

- [SHOULD] **チャートのバージョニング**: 小さな修正はパッチバージョン、新機能や機能拡張はマイナーバージョン、破壊的変更はメジャーバージョンとし、同じバージョンの再利用を防ぐ。

- [SHOULD] **ライブラリチャートや依存関係**: 共通パターンは Helm ライブラリチャートやベースチャートを使用して再利用する。ただし過度な抽象化は避ける。

- [MUST] **サブチャートの重複を避ける**: 依存チャートを利用する場合、リソース名や値の競合を避けるために `.Release.Name` やチャート名を含めるなどして調整すること。

- [SHOULD] **Helm リリース名**: 固定のリリース名で動作するように設計する。リソース名に `{{ .Release.Name }}` を含め、一意性を保証する。

- [MUST] **チャートの使用方法を文書化**すること。README.md や社内ドキュメントに設定可能な値や使用方法を明記する。

- [SHOULD] **チャートのテスト**を実施すること。Helm test フックや helm unittest を用いてレンダリング結果を検証するか、最低でも開発用クラスターで実際の動作を確認する。

---

## GitOps ワークフローとデプロイメント

- [MUST] **コードとしての設定管理**: Kubernetes マニフェスト、Helm リリース仕様などは全て Git などで管理し、手動変更を避けること。これにより履歴管理、監査、ロールバックが可能となる。

- [SHOULD] GitOps リポジトリを環境ごとに明確に整理する。例：

  ```text
  cluster-prod/
    namespace-a/
      app1/
        release.yaml
      app2/
        ...
  cluster-staging/
    ...
  ```

  または環境別ブランチを使用する。

- [MUST] **プルリクエスト（Pull Request）レビュー**: Kubernetes 設定の変更は PR で実施し、他者レビューを経ること。直接の main や環境ブランチへのプッシュは禁止する。

- [MUST] **CI による検証**: YAML リンティング、スキーマ検証（`kubectl apply --dry-run=server`、kubeval）、OPA Conftest 等を使用し、設定の正当性を自動検証する。Helm は `helm lint`、`helm template` で確認する。

- [SHOULD] **GitOps による継続的デプロイメント**: Argo CD や Flux を使い Git リポジトリをクラスターと同期させ、構成のドリフトを防止すること。

- [MUST] **ドリフト検出**: GitOps オペレーターが無い場合でも、定期的に実際の状態と Git を比較し、設定ドリフトを検出・修正すること。

- [MUST] **環境間のプロモーション**: 開発→ステージング→本番のプロモーションプロセスを明確にする。環境固有の違い以外は、検証済みの同一アーティファクトを展開する。

- [SHOULD] 不変なアーティファクト（イメージタグ、Helm チャートバージョン）を明確に指定し、環境間で同一のアーティファクトを使用すること。

- [MUST] **ロールバック戦略**: コミットのリバートによる迅速なロールバックを可能にし、安定した状態にはタグを付けること。

- [SHOULD] **段階的なロールアウト** をサポートする。カナリアリリースやブルーグリーンデプロイメントをGitOpsで実施できるように設計する。

- [SHOULD] **変更ログの記録** を行い、誰がどの変更を行ったかを監査可能にすること。

- [MUST] **本番設定変更へのアクセス制御** を厳格に管理し、承認者を限定すること。

- [MUST] **Git にシークレット情報を保存しない**: 暗号化された SealedSecrets や外部管理のシークレットのみを利用し、プレーンテキストで保存してはならない。

- [MUST] GitOps リポジトリを常に整理し、不要になった設定やNamespaceを削除またはアーカイブすること。

- [SHOULD] GitOps プロセスの文書化とオンボーディング資料を整備し、新規メンバーがプロセスを迅速に理解できるようにする。

修正後の文章を示す：

---

## 本番運用の準備と可観測性

- [MUST] **ヘルスチェック**:
すべての本番ワークロードに readiness および liveness プローブを実装すること（[Amazon EKS Governance | Searce](https://blog.searce.com/amazon-eks-governance-mitigating-risks-and-meeting-standards-using-opa-gatekeeper-ab4f7756d351#:~:text=,Enforcing%20Pod%20Security%20Context)）。
readiness プローブが失敗すると Pod はトラフィックから外され、liveness プローブが失敗すると再起動される。
アプリケーションは依存サービス（DB等）のヘルスチェックを含むエンドポイントを用意する。

- [SHOULD] **グレースフルシャットダウン**:
Pod は SIGTERM を受けたら適切にシャットダウンすること。
`terminationGracePeriodSeconds` を適切に設定し、新規リクエストを停止（readiness を false に変更）して、進行中の処理を完了する。Web アプリは30秒以上の猶予を推奨する。

- [SHOULD] **PodDisruptionBudget (PDB)** の利用:
ノードのメンテナンス時に重要サービスの Pod が同時に停止しないよう PDB を設定する。運用チームと連携し、クリティカルサービスで適切な PDB を確保すること。

- [MUST] **ログ出力**:
すべてのアプリケーションはログを STDOUT/STDERR（またはサイドカーが取得するファイル）に出力すること。ログは EFK スタックや Cloud Logging 等で収集可能な構造化フォーマット（JSON等）であること。
好ましいログの例:
```json
{"timestamp":"2025-03-27T13:45:00Z","level":"INFO","msg":"Payment processed","orderId":1234}
```

- [MUST] **ログに機密情報を記録しない**:
パスワードや個人情報がログに出力されないよう、コードレビューやスキャンツールで管理する。
やむを得ない場合はマスキングやハッシュ化を行う。

- [SHOULD] **ログの冗長性**:
本番では INFO を標準とし、問題発生時は WARN、エラー時は ERROR を使い分ける。DEBUG レベルの詳細ログは通常は無効にし、必要時のみ設定で有効化する仕組みを提供する。

- [MUST] **モニタリングとメトリクス**:
すべてのサービスはアプリ固有メトリクス（HTTP リクエスト数、レイテンシ、エラー率）を Prometheus 等で収集可能にすること。
Pod への設定例:
```yaml
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8080"
  prometheus.io/path: "/metrics"
```

- [SHOULD] **ダッシュボードとアラート**:
基本的な指標（エラー率、CPU・メモリ利用率等）を可視化するダッシュボードを提供すること。
高エラー率、リソースの逼迫（90%以上）、Pod の再起動頻度等、重要条件でアラートを発砲し、PagerDuty 等のインシデント管理と連携する。
可能ならばサービスレベル目標（SLO）を定義して達成状況をモニターする。

- [SHOULD] **分散トレーシング**:
必要に応じて OpenTelemetry や Jaeger を利用し、トレース ID をリクエストに伝播させ、問題追跡を容易にする。
複雑なマイクロサービス環境では特に推奨される。実装時はトレースのサンプリング率に注意すること。

- [MUST] **バックアップおよびリカバリ**:
ステートフルコンポーネント（DB、PersistentVolume）はバックアップを策定する。
定期的なバックアップのスケジュール化や、外部ストレージへの保存を実施し、リカバリ手順を明確に文書化すること。

- [SHOULD] **耐障害性テスト**:
非本番環境でカオスエンジニアリングを定期的に実施し、Pod 障害等を注入してシステムの回復性を検証する。
必要に応じてスケーリング設定やレプリカ数を調整する。

- [MUST] **クリティカルなコンポーネントの高可用性**:
重要なサービスには単一障害点を排除する設計が必要である。
Kafka や ZooKeeper のような StatefulSet はレプリカを奇数個設定し、anti-affinity を活用して複数ノードに分散する。
外部サービスが停止・遅延した際のタイムアウト、リトライ、フォールバックをコードで実装し、依存関係の障害に耐えられるように設計する。

- [SHOULD] **キャパシティプランニング**:
継続的なリソース利用状況の監視を行い、長期的な負荷増に対応可能なキャパシティを維持すること。
CPU 利用率が常時 70% を超える場合、ノード追加やリソース増強を検討する。

- [MUST] **インシデント対応**:
ノード障害、Pod クラッシュ、設定ロールバック、シークレットローテーションなどのランブックを用意し、インシデント発生時の対応を明確化する。
名前・ラベルの整備、ログ・メトリクスとの連携により迅速に対象を特定できる体制を整える。定期的な訓練を実施し、対応能力を維持すること。
