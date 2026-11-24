<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { causeOf, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  let { ticket }: { ticket: number } = $props();

  // Three problems: the desk's own cause rule, with the one-day moves taken out of `parser`.
  const merged = $derived(uniqueById(reports as Report[]).filter((r) => r.mergedInto === ticket));
  const problemOf = (r: Report) => (notes.oneDayMove.includes(r.id) ? 'oneDay' : (causeOf(r, tenantOf(r)) ?? 'unplaced'));
  const rows = $derived(
    ['parser', 'hint', 'oneDay', 'unplaced']
      .map((p) => {
        const rs = merged.filter((r) => problemOf(r) === p);
        const days = rs.map((r) => r.receivedAt.slice(0, 10)).sort();
        return {
          problem: p,
          n: rs.length,
          tenants: new Set(rs.map((r) => r.tenant)).size,
          first: days[0],
          last: days[days.length - 1],
          // Intake has no tag for a one-day move, so none of those can be tagged right.
          taggedRight: p === 'oneDay' ? 0 : rs.filter((r) => r.meta.intakeTag === p).length,
        };
      })
      .filter((x) => x.n > 0),
  );
</script>

<p><strong>{merged.length}</strong> reports merged into #{ticket} · <strong>{rows.length}</strong> problems</p>

<table>
  <thead><tr><th>problem</th><th>reports</th><th>customers</th><th>seen</th><th>intake tagged it right</th></tr></thead>
  <tbody>
    {#each rows as x (x.problem)}
      <tr><td><code>{x.problem}</code></td><td>{x.n}</td><td>{x.tenants}</td><td>{x.first} → {x.last}</td><td>{x.taggedRight} of {x.n}</td></tr>
    {/each}
  </tbody>
</table>
