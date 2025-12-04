<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>Cause-Area Matchmaking — Interactive Map</title>
  <meta name="viewport" content="width=device-width,initial-scale=1">
  <style>
    body { font-family: Arial, Helvetica, sans-serif; margin:0; }
    #controls { width: 320px; position: absolute; left: 12px; top: 12px; background: rgba(255,255,255,0.96); border-radius:8px; padding:12px; box-shadow:0 4px 14px rgba(0,0,0,0.12); z-index:10; }
    #chart { width: 100%; height: 100vh; }
    .node { stroke: #222; stroke-width:0.6px; cursor:pointer; }
    .legend-item { display:flex; align-items:center; margin-bottom:6px; font-size:13px; }
    .color-swatch { width:16px; height:16px; border-radius:3px; margin-right:8px; border:1px solid #333; }
    .tooltip { position: absolute; pointer-events: none; background: #fff; border:1px solid #ccc; padding:10px; border-radius:6px; font-size:13px; box-shadow:0 6px 18px rgba(0,0,0,0.12); max-width:320px; }
    .search { width:100%; padding:6px 8px; margin-bottom:8px; border-radius:6px; border:1px solid #ddd; }
    .filter { width:100%; padding:6px 8px; margin-bottom:8px; border-radius:6px; border:1px solid #ddd; }
    .small { font-size:12px; color:#666; }
    .btn { padding:8px 10px; border-radius:6px; background:#1976d2; color:#fff; border:none; cursor:pointer; display:inline-block; margin-right:6px; }
    .btn:active { transform: translateY(1px); }
    #infoPanel { position:absolute; right:12px; top:12px; width:320px; max-height:80vh; overflow:auto; background:rgba(255,255,255,0.98); padding:12px; border-radius:8px; box-shadow:0 4px 14px rgba(0,0,0,0.12); z-index:10; }
    #infoPanel h3 { margin:0 0 6px 0; }
    #infoPanel p { margin:6px 0; }
  </style>
</head>
<body>

<div id="controls">
  <input id="search" class="search" placeholder="Search organization..." />
  <select id="causeFilter" class="filter">
    <option value="__all__">All cause areas</option>
  </select>
  <div style="margin-bottom:8px;">
    <button id="reset" class="btn">Reset view</button>
    <button id="exportMatches" class="btn" style="background:#388e3c;">Export visible matches</button>
  </div>
  <div id="legendTitle" class="small" style="margin-bottom:6px;">Legend (click a cause to filter)</div>
  <div id="legend"></div>
  <div class="small" style="margin-top:8px;">Tip: hover nodes to see details; click a node to lock it in the side panel.</div>
</div>

<div id="infoPanel" style="display:none;"></div>
<svg id="chart"></svg>
<div id="tooltip" class="tooltip" style="display:none;"></div>

<!-- D3 -->
<script src="https://d3js.org/d3.v7.min.js"></script>
<script>

(async function() {
  // CONFIG: filenames (must be in same folder as HTML)
  const NODES_CSV = 'cleaned_orgs.csv';
  const LINKS_CSV = 'org_matches.csv';

  // load data
  const nodesRaw = await d3.csv(NODES_CSV);
  const linksRaw = await d3.csv(LINKS_CSV);

  // normalize and build structures
  // nodesRaw expected columns: Organization,PrimaryCause,Needs,Offers,Asks,Employees
  // linksRaw expected columns: From,To,MatchedAreas

  // create nodes map
  const nodesMap = new Map();
  nodesRaw.forEach(r=>{
    const id = r.Organization;
    nodesMap.set(id, {
      id,
      primaryCause: r.PrimaryCause || 'Unspecified',
      needs: r.Needs ? r.Needs.split(';').map(s=>s.trim()).filter(Boolean) : [],
      offers: r.Offers ? r.Offers.split(';').map(s=>s.trim()).filter(Boolean) : [],
      asks: r.Asks ? r.Asks.split(';').map(s=>s.trim()).filter(Boolean) : [],
      employees: +r.Employees || 1
    });
  });

  // build nodes array and links array
  const nodes = Array.from(nodesMap.values());
  const nodeIndex = new Map(nodes.map((n,i)=>[n.id,i]));

  const links = linksRaw.map(l=>{
    const source = l.From;
    const target = l.To;
    const areas = l.MatchedAreas ? l.MatchedAreas.split(';').map(s=>s.trim()).filter(Boolean) : [];
    return {
      source, target, areas
    };
  }).filter(l => nodeIndex.has(l.source) && nodeIndex.has(l.target));

  // compute unique cause areas
  const causes = Array.from(new Set(nodes.map(d=>d.primaryCause || 'Unspecified'))).sort();

  // color scale
  const color = d3.scaleOrdinal(d3.schemeTableau10).domain(causes);

  // populate cause filter and legend
  const causeFilter = d3.select('#causeFilter');
  causes.forEach(c=>{
    causeFilter.append('option').attr('value', c).text(c);
  });

  const legend = d3.select('#legend');
  causes.forEach(c=>{
    const item = legend.append('div').attr('class','legend-item').style('cursor','pointer')
      .on('click', ()=>{
        causeFilter.property('value', c);
        applyFilters();
      });
    item.append('div').attr('class','color-swatch').style('background', color(c));
    item.append('div').text(`${c} (${nodes.filter(n=>n.primaryCause===c).length})`);
  });

  // SVG setup
  const svg = d3.select('#chart');
  const width = window.innerWidth;
  const height = window.innerHeight;
  svg.attr('width', width).attr('height', height);

  const g = svg.append('g');

  // zoom
  const zoom = d3.zoom().scaleExtent([0.2, 3]).on('zoom', (event)=> g.attr('transform', event.transform));
  svg.call(zoom);

  // build simulation
  const simulation = d3.forceSimulation(nodes)
    .force('link', d3.forceLink(links).id(d=>d.id).distance(120).strength(0.6))
    .force('charge', d3.forceManyBody().strength(-120))
    .force('center', d3.forceCenter(width/2, height/2))
    .force('collide', d3.forceCollide().radius(d => 20 + Math.sqrt(d.employees) * 6));

  // link elements
  const linkG = g.append('g').attr('class','links');
  const link = linkG.selectAll('path').data(links).enter().append('path')
    .attr('stroke', '#999').attr('stroke-opacity', 0.4).attr('fill','none')
    .attr('marker-end', 'url(#arrow)');

  // arrow marker
  svg.append('defs').append('marker')
    .attr('id','arrow')
    .attr('viewBox','0 -5 10 10')
    .attr('refX',14)
    .attr('refY',0)
    .attr('markerWidth',6).attr('markerHeight',6)
    .attr('orient','auto')
    .append('path').attr('d','M0,-5L10,0L0,5').attr('fill','#777');

  // node elements
  const nodeG = g.append('g').attr('class','nodes');
  const node = nodeG.selectAll('g.node').data(nodes).enter().append('g').attr('class','node');

  node.append('circle')
    .attr('r', d => 8 + Math.sqrt(d.employees) * 3)
    .attr('fill', d => color(d.primaryCause))
    .attr('stroke', '#222').attr('stroke-width', 0.6)
    .on('mouseover', handleMouseOver)
    .on('mouseout', handleMouseOut)
    .on('click', handleClick);

  node.append('text')
    .text(d => d.id)
    .attr('font-size', 10)
    .attr('dx', 12)
    .attr('dy', 4)
    .style('pointer-events','none')
    .each(wrapText);

  // tooltip and info panel
  const tooltip = d3.select('#tooltip');
  const infoPanel = d3.select('#infoPanel');

  // interactivity: search + filter buttons
  d3.select('#search').on('input', applyFilters);
  d3.select('#causeFilter').on('change', applyFilters);
  d3.select('#reset').on('click', ()=>{ d3.select('#search').property('value',''); d3.select('#causeFilter').property('value','__all__'); applyFilters(); zoom.transform(svg, d3.zoomIdentity); });
  d3.select('#exportMatches').on('click', exportVisibleMatches);

  // simulation tick
  simulation.on('tick', ()=>{
    // curved link path
    link.attr('d', d=>{
      const s = d.source;
      const t = d.target;
      if (!s.x || !t.x) return '';
      // compute unit vector and perpendicular for curve
      const dx = t.x - s.x, dy = t.y - s.y;
      const dr = Math.sqrt(dx*dx + dy*dy) * 1.4;
      // Use simple quadratic Bezier
      return `M${s.x},${s.y} Q${(s.x + t.x)/2 + (dy/dr)*40},${(s.y + t.y)/2 - (dx/dr)*40} ${t.x},${t.y}`;
    });

    node.attr('transform', d => `translate(${d.x},${d.y})`);
  });

  // helpers
  function wrapText(d) {
    // shorten labels visually (only first ~28 chars)
    const el = d3.select(this);
    const text = d.id.length > 24 ? (d.id.slice(0,24) + '…') : d.id;
    el.text(text);
  }

  function handleMouseOver(event, d) {
    // highlight node & connected links
    d3.select(this).attr('stroke-width', 1.8);
    tooltip.style('display','block');
    const html = tooltipHtml(d);
    tooltip.html(html);
    moveTooltip(event);
    // emphasize links
    link.attr('stroke-opacity', l => (l.source.id === d.id || l.target.id === d.id) ? 0.9 : 0.08)
        .attr('stroke', l => (l.source.id === d.id || l.target.id === d.id) ? '#333' : '#bbb');
  }

  function handleMouseOut(event, d) {
    d3.select(this).attr('stroke-width', 0.6);
    tooltip.style('display','none');
    link.attr('stroke-opacity', 0.4).attr('stroke','#999');
  }

  function handleClick(event, d) {
    // show info panel with detailed data and top partner suggestions
    const incoming = links.filter(l => l.target === d.id).map(l => ({from:l.source, areas:l.areas}));
    const outgoing = links.filter(l => l.source === d.id).map(l => ({to:l.target, areas:l.areas}));

    const html = [
      `<h3>${escapeHtml(d.id)}</h3>`,
      `<p class="small"><strong>Primary cause:</strong> ${escapeHtml(d.primaryCause)}</p>`,
      `<p class="small"><strong>Employees:</strong> ${d.employees}</p>`,
      `<p class="small"><strong>Needs:</strong> ${escapeHtml(d.needs.join('; ') || '—')}</p>`,
      `<p class="small"><strong>Offers:</strong> ${escapeHtml(d.offers.join('; ') || '—')}</p>`,
      `<p class="small"><strong>Asks:</strong> ${escapeHtml(d.asks.join('; ') || '—')}</p>`,
      `<hr />`,
      `<h4>Top organizations that CAN support this org (incoming)</h4>`,
      (incoming.length ? '<ul>' + incoming.slice(0,10).map(i => `<li><strong>${escapeHtml(i.from)}</strong> — ${escapeHtml(i.areas.join('; '))}</li>`).join('') + '</ul>' : '<p class="small">No direct matches found.</p>'),
      `<h4>Organizations this org can support (outgoing)</h4>`,
      (outgoing.length ? '<ul>' + outgoing.slice(0,10).map(i => `<li><strong>${escapeHtml(i.to)}</strong> — ${escapeHtml(i.areas.join('; '))}</li>`).join('') + '</ul>' : '<p class="small">No direct matches found.</p>')
    ].join('');

    infoPanel.style('display', 'block').html(html);
    // center on node (smooth zoom)
    const scale = 1.5;
    const x = d.x, y = d.y;
    const translate = [width/2 - scale * x, height/2 - scale * y];
    svg.transition().duration(750).call(zoom.transform, d3.zoomIdentity.translate(translate[0], translate[1]).scale(scale));
  }

  function moveTooltip(event) {
    const [mx, my] = d3.pointer(event);
    tooltip.style('left', (mx + 18) + 'px').style('top', (my + 18) + 'px');
  }

  function tooltipHtml(d) {
    return `<strong>${escapeHtml(d.id)}</strong><br/>
            <small><strong>Cause:</strong> ${escapeHtml(d.primaryCause)}</small><br/>
            <small><strong>Employees:</strong> ${d.employees}</small><br/>
            <small><strong>Needs:</strong> ${escapeHtml(d.needs.join('; ') || '—')}</small><br/>
            <small><strong>Offers:</strong> ${escapeHtml(d.offers.join('; ') || '—')}</small>`;
  }

  // escapes
  function escapeHtml(str){ if(!str) return ''; return str.replace(/[&<>"']/g, s => ({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'}[s])); }

  // filtering
  function applyFilters() {
    const q = d3.select('#search').property('value').toLowerCase().trim();
    const selectedCause = d3.select('#causeFilter').property('value');

    node.style('display', d => {
      const bySearch = q === '' || d.id.toLowerCase().includes(q) || (d.primaryCause && d.primaryCause.toLowerCase().includes(q));
      const byCause = (selectedCause === '__all__') || (d.primaryCause === selectedCause);
      return (bySearch && byCause) ? 'inline' : 'none';
    });

    // hide links if either endpoint hidden
    const visibleSet = new Set(nodes.filter(d => {
      const el = d3.selectAll('g.node').filter(nd => nd.id === d.id);
      return el.size() && d3.select(el.node()).style('display') !== 'none';
    }).map(d=>d.id));

    link.style('display', l => (visibleSet.has(l.source) && visibleSet.has(l.target)) ? 'inline' : 'none');
  }

  // export visible matches to CSV
  function exportVisibleMatches() {
    // collect currently visible links
    const visibleLinks = links.filter(l=>{
      const nodeA = nodesMapGet(l.source);
      const nodeB = nodesMapGet(l.target);
      if (!nodeA || !nodeB) return false;
      const aVisible = d3.selectAll('g.node').filter(d=>d.id===nodeA.id).style('display') !== 'none';
      const bVisible = d3.selectAll('g.node').filter(d=>d.id===nodeB.id).style('display') !== 'none';
      return aVisible && bVisible;
    });
    if (visibleLinks.length === 0) { alert('No visible matches to export.'); return; }
    const csv = ['From,To,MatchedAreas'].concat(visibleLinks.map(l => `"${l.source.replace(/"/g,'""')}","${l.target.replace(/"/g,'""')}","${(l.areas||[]).join('; ').replace(/"/g,'""')}"`)).join('\n');
    const blob = new Blob([csv], {type: 'text/csv;charset=utf-8;'});
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a'); a.href = url; a.download = 'visible_matches.csv'; document.body.appendChild(a); a.click(); a.remove(); URL.revokeObjectURL(url);
  }

  // helper: lookup node record
  function nodesMapGet(id) {
    return nodes.find(n => n.id === id);
  }

  // responsive
  window.addEventListener('resize', ()=> {
    const w = window.innerWidth, h = window.innerHeight;
    svg.attr('width', w).attr('height', h);
    simulation.force('center', d3.forceCenter(w/2, h/2));
    simulation.alpha(0.3).restart();
  });

  // initial layout nudge
  simulation.alpha(1).restart();

})(); // end IIFE

</script>
</body>
</html>
