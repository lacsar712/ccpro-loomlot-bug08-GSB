<script>
  import { onMount } from 'svelte';
  import { api, toLocalInput, fromLocalInput } from '../lib/api.js';

  let lots = [];
  let rows = [];
  let error = '';
  let form = {
    dyeLotId: '',
    checkedAt: toLocalInput(new Date().toISOString()),
    washFastness: 4,
    rubFastness: 3.5,
    tempC: 40,
    notes: '',
  };
  let editing = null;

  // '' 表示不按染程过滤（全部染程）
  let filterLotId = '';
  let page = 1;
  const pageSize = 10;
  let total = 0;
  $: totalPages = Math.max(1, Math.ceil(total / pageSize));

  async function load() {
    error = '';
    try {
      lots = await api('/dye-lots');
      if (!form.dyeLotId && lots.length) form.dyeLotId = String(lots[0].id);
      const params = new URLSearchParams({ page: String(page), pageSize: String(pageSize) });
      if (filterLotId) params.set('dyeLotId', filterLotId);
      const res = await api('/fastness-checks?' + params.toString());
      rows = res.items || [];
      total = res.total || 0;
      page = res.page || page;
      // 删除后当前页可能已空，回到第一页
      if (rows.length === 0 && page > 1) {
        page = 1;
        await load();
      }
    } catch (e) {
      error = e.message;
    }
  }

  onMount(load);

  function changeFilter() {
    page = 1;
    load();
  }

  function goPage(p) {
    if (p < 1 || p > totalPages || p === page) return;
    page = p;
    load();
  }

  function lotLabel(id) {
    const lot = lots.find((x) => x.id === id);
    return lot ? `${lot.recipeName} (#${lot.id})` : id;
  }

  async function save() {
    error = '';
    try {
      const body = {
        dyeLotId: Number(form.dyeLotId),
        checkedAt: fromLocalInput(form.checkedAt),
        washFastness: Number(form.washFastness),
        rubFastness: Number(form.rubFastness),
        tempC: Number(form.tempC),
        notes: form.notes.trim() || null,
      };
      if (editing) {
        await api(`/fastness-checks/${editing}`, { method: 'PUT', body: JSON.stringify(body) });
      } else {
        await api('/fastness-checks', { method: 'POST', body: JSON.stringify(body) });
      }
      editing = null;
      form = {
        ...form,
        checkedAt: toLocalInput(new Date().toISOString()),
        notes: '',
      };
      await load();
    } catch (e) {
      error = e.message;
    }
  }

  function startEdit(row) {
    editing = row.id;
    form = {
      dyeLotId: String(row.dyeLotId),
      checkedAt: toLocalInput(row.checkedAt),
      washFastness: row.washFastness,
      rubFastness: row.rubFastness,
      tempC: row.tempC,
      notes: row.notes || '',
    };
  }

  async function remove(id) {
    if (!confirm('确认删除该抽检？')) return;
    error = '';
    try {
      await api(`/fastness-checks/${id}`, { method: 'DELETE' });
      await load();
    } catch (e) {
      error = e.message;
    }
  }
</script>

<h1 class="page-title">色牢度抽检</h1>
<p class="page-sub">耐洗 1–5 级；摩擦牢度须大于 0；记录检测温度。</p>

<div class="panel" style="margin-bottom:1rem;">
  <div class="form-grid">
    <label
      >染程
      <select bind:value={form.dyeLotId}>
        {#each lots as lot}
          <option value={String(lot.id)}>{lot.recipeName} · {lot.fabricKg}kg</option>
        {/each}
      </select>
    </label>
    <label>检测时间 <input type="datetime-local" bind:value={form.checkedAt} /></label>
    <label>耐洗 (1–5) <input type="number" min="1" max="5" bind:value={form.washFastness} /></label>
    <label>摩擦 (&gt;0) <input type="number" step="0.1" min="0.1" bind:value={form.rubFastness} /></label>
    <label>温度 ℃ <input type="number" step="0.1" bind:value={form.tempC} /></label>
    <label>备注 <input bind:value={form.notes} /></label>
  </div>
  <div class="toolbar">
    <button class="btn" type="button" on:click={save}>{editing ? '保存修改' : '登记抽检'}</button>
    {#if editing}
      <button class="btn ghost" type="button" on:click={() => (editing = null)}>取消</button>
    {/if}
  </div>
  {#if error}<p class="err">{error}</p>{/if}
</div>

<div class="panel">
  <div class="toolbar" style="margin-bottom:0.75rem;">
    <label style="display:flex;align-items:center;gap:0.5rem;margin:0;">
      染程筛选
      <select bind:value={filterLotId} on:change={changeFilter}>
        <option value="">全部染程</option>
        {#each lots as lot}
          <option value={String(lot.id)}>{lot.recipeName} (#{lot.id})</option>
        {/each}
      </select>
    </label>
    <span style="color:var(--indigo-mist);font-size:0.9rem;">共 {total} 条</span>
  </div>
  <table>
    <thead>
      <tr>
        <th>ID</th>
        <th>染程</th>
        <th>检测时间</th>
        <th>耐洗</th>
        <th>摩擦</th>
        <th>温度</th>
        <th>备注</th>
        <th></th>
      </tr>
    </thead>
    <tbody>
      {#each rows as row}
        <tr>
          <td>{row.id}</td>
          <td>{lotLabel(row.dyeLotId)}</td>
          <td>{new Date(row.checkedAt).toLocaleString()}</td>
          <td>{row.washFastness}</td>
          <td>{row.rubFastness}</td>
          <td>{row.tempC}℃</td>
          <td>{row.notes || '—'}</td>
          <td class="row-actions">
            <button class="btn ghost small" type="button" on:click={() => startEdit(row)}>编辑</button>
            <button class="btn danger small" type="button" on:click={() => remove(row.id)}>删除</button>
          </td>
        </tr>
      {/each}
      {#if rows.length === 0}
        <tr><td colspan="8" style="text-align:center;color:var(--indigo-mist);">该染程下暂无抽检记录</td></tr>
      {/if}
    </tbody>
  </table>
  {#if totalPages > 1}
    <div class="toolbar" style="justify-content:flex-end;margin-bottom:0;">
      <button class="btn ghost small" type="button" disabled={page <= 1} on:click={() => goPage(page - 1)}>上一页</button>
      <span style="color:var(--indigo-mist);font-size:0.9rem;">第 {page} / {totalPages} 页</span>
      <button
        class="btn ghost small"
        type="button"
        disabled={page >= totalPages}
        on:click={() => goPage(page + 1)}>下一页</button>
    </div>
  {/if}
</div>
