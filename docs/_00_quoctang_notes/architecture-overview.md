# openclaw — Architecture Overview

> Notes cá nhân, không phải official docs.
> Cập nhật: 2026-04-22

## Luồng tổng quát

```
User/Client
    │
    ▼
[Gateway Protocol]      src/gateway/protocol/   ← wire contract operator ↔ node
    │
    ▼
[Gateway Server]        src/gateway/            ← hot path, lightweight artifacts
    │
    ├──► [Channels]     src/channels/           ← core channel impl (PRIVATE)
    │         │
    │         ▼
    │    [Plugin SDK]   src/plugin-sdk/         ← public contract cho plugin authors
    │         │
    ▼         ▼
[Plugin Loader]         src/plugins/            ← discover, validate manifest, lazy load
    │
    ▼
[Extensions]            extensions/             ← bundled plugins (telegram, openai...)
```

## Từng vùng

### `src/gateway/protocol/`
- Wire protocol giữa gateway ↔ operator clients/nodes
- **Schema = contract**: thêm field phải additive, không xóa/đổi type
- Đổi breaking → cần versioning + docs + client follow-through

### `src/gateway/`
- Gateway server, được xem là hot path
- Không load full plugin runtime khi chỉ cần descriptor tĩnh
- Dùng "lightweight artifacts" (resolver tĩnh) thay full plugin load
- Test: disable schedulers/pollers, reuse suite-level servers

### `src/channels/`
- Core channel implementation — **nội bộ, không phải public API**
- Plugin authors **không được** import trực tiếp từ đây
- Plugin chỉ access qua seam trong Plugin SDK
- Entrypoints: `channel.ts`, `shared.ts`, `setup.ts`, `gateway.ts`, `outbound.ts`
- Async surfaces phải lazy

### `src/plugin-sdk/`
- **Public contract duy nhất** giữa plugin và core
- Cả bundled plugin cũng chỉ được import từ đây
- Module load phải rẻ (không execute setup code lúc import)
- Prefer narrow subpaths over broad barrels
- Có API baselines — breaking changes cần versioning

### `src/plugins/`
- Plugin discovery, manifest validation, loading, registry assembly
- **Manifest-first**: mọi behavior khai báo trong manifest, không hardcode trong core
- Lazy discovery — không materialize toàn bộ plugin lúc startup
- Control-plane vs runtime-plane tách biệt rõ ràng

### `extensions/`
- Bundled plugins, chơi theo đúng rule plugin bên thứ 3
- Chỉ import từ `openclaw/plugin-sdk/*` và local barrels (`api.ts`, `runtime-api.ts`)
- **Không** import `src/**` core, không import extension khác
- Không seeding global registry từ bundled plugins

### `src/agents/`
- Tests cho AI agent infrastructure
- Test chậm = tín hiệu architecture bị expose quá nhiều
- Dùng lightweight artifacts cho schema/discovery thay full runtime load

### `ui/`
- Control UI
- i18n: chỉ sửa English source → `pnpm ui:i18n:sync` → auto-gen các locale khác
- Không hand-edit non-English locale files

### `docs/`
- Mintlify hosting
- Root-relative links, không có `.md` suffix
- Thứ tự alphabetical trừ khi mô tả runtime order

### `scripts/`
- Wrappers cho test/lint/typecheck: `run-vitest.mjs`, `run-oxlint.mjs`, `run-tsgo.mjs`
- Dùng wrapper có sẵn, không gọi raw `vitest`/`tsc`

### `test/helpers/`
- Shared test helpers dùng chung giữa core và extension tests
- Dùng `bundled-plugin-public-surface.ts` loader, không deep-mock plugin internals

## Nguyên tắc kiến trúc cốt lõi

1. **Core extension-agnostic**: không có special case trong core cho bundled plugin cụ thể
2. **Manifest-first**: behavior khai báo trong manifest, không ẩn trong code
3. **Lazy loading**: không materialize plugin lúc không cần
4. **Lightweight artifacts**: dùng static descriptor trước, load full runtime khi cần
5. **Control-plane vs runtime-plane**: tách biệt rõ ràng
6. **Backwards-compatible seams**: additive trước, breaking cần versioning

## Key commands

```bash
pnpm check:changed      # typecheck + lint + test cho phần đã thay đổi
pnpm test               # full test suite
pnpm build              # build (bắt buộc nếu đụng module boundaries)
pnpm docs:list          # xem catalog docs
pnpm config:docs:gen    # generate config docs
pnpm plugin-sdk:api:gen # generate SDK API baseline
```
