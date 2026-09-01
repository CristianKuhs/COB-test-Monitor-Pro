mport { useEffect, useState } from 'react';
import { base44 } from '@/api/base44Client';
import { fetchAllMonitorings, fetchActiveAgents, channelLabel, monthKey, monthLabel, avg, CRITERIA_DEFAULT } from '@/lib/monitoring';
import { getClassificationRanges, classifyScore } from '@/lib/adminAuth';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select';
import { Trophy, ClipboardCheck, TrendingUp, Award, Phone, MessageSquare } from 'lucide-react';
import {
  LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, ResponsiveContainer,
  BarChart, Bar, PieChart, Pie, Cell, Legend,
} from 'recharts';

export default function Dashboard() {
  const [monitorings, setMonitorings] = useState(null);
  const [ranges, setRanges] = useState([]);

  useEffect(() => {
    (async () => {
      const [m, r] = await Promise.all([fetchAllMonitorings(), getClassificationRanges()]);
      setMonitorings(m);
      setRanges(r);
    })();
  }, []);

  if (!monitorings) return <Loading />;

  const total = monitorings.length;
  const scores = monitorings.map((m) => m.total_score || 0);
  const generalAvg = avg(scores);
  const effectiveness = generalAvg;

  // best agent
  const byAgent = {};
  monitorings.forEach((m) => {
    if (!byAgent[m.agent_name]) byAgent[m.agent_name] = [];
    byAgent[m.agent_name].push(m.total_score || 0);
  });
  let bestAgent = { name: '—', avg: 0 };
  Object.entries(byAgent).forEach(([name, arr]) => {
    const a = avg(arr);
    if (a > bestAgent.avg) bestAgent = { name, avg: a };
  });

  // best channel
  const byChannel = { ligacao: [], chat: [] };
  monitorings.forEach((m) => { if (byChannel[m.channel]) byChannel[m.channel].push(m.total_score || 0); });
  let bestChannel = { key: '—', avg: 0 };
  Object.entries(byChannel).forEach(([k, arr]) => {
    if (!arr.length) return;
    const a = avg(arr);
    if (a > bestChannel.avg) bestChannel = { key: k, avg: a };
  });

  // monthly evolution
  const byMonth = {};
  monitorings.forEach((m) => {
    const k = monthKey(m.audit_date);
    if (!k) return;
    if (!byMonth[k]) byMonth[k] = [];
    byMonth[k].push(m.total_score || 0);
  });
  const evolution = Object.keys(byMonth).sort().map((k) => ({
    month: monthLabel(k).split(' / ')[0],
    monitorias: byMonth[k].length,
    media: avg(byMonth[k]),
  }));

  // monitorias por agente
  const perAgent = Object.entries(byAgent).map(([name, arr]) => ({ name, total: arr.length })).sort((a, b) => b.total - a.total);

  // distribuicao por canal
  const channelDist = [
    { name: 'Ligação', value: monitorings.filter((m) => m.channel === 'ligacao').length },
    { name: 'Chat', value: monitorings.filter((m) => m.channel === 'chat').length },
  ].filter((d) => d.value > 0);

  // desempenho dos criterios
  const critPerf = CRITERIA_DEFAULT.map((desc, i) => {
    const vals = monitorings
      .map((m) => m.criteria_scores?.[i])
      .filter((v) => typeof v === 'number');
    return { desc: `C${i + 1}`, media: vals.length ? avg(vals) : 0 };
  });

  return (
    <div className="space-y-6">
      <div>
        <h1 className="font-heading text-2xl font-bold">Dashboard</h1>
        <p className="text-sm text-muted-foreground">Visão executiva das monitorias</p>
      </div>
      <div className="grid gap-4 grid-cols-2 md:grid-cols-3 xl:grid-cols-5">
        <StatCard icon={ClipboardCheck} label="Monitorias realizadas" value={total} accent="primary" />
        <StatCard icon={TrendingUp} label="Média geral" value={`${generalAvg}/100`} accent="brand" />
        <StatCard icon={Award} label="Efetividade" value={`${effectiveness}%`} accent="brand" />
        <StatCard icon={Trophy} label="Melhor agente" value={bestAgent.name} sub={`${bestAgent.avg}/100`} accent="primary" />
        <StatCard
          icon={bestChannel.key === 'chat' ? MessageSquare : Phone}
          label="Melhor canal"
          value={bestChannel.key === 'chat' ? 'Chat' : bestChannel.key === 'ligacao' ? 'Ligação' : '—'}
          sub={bestChannel.avg ? `${bestChannel.avg}%` : ''}
          accent="primary"
        />
      </div>
      <div className="grid gap-6 md:grid-cols-2">
        <Card>
          <CardHeader><CardTitle className="text-base">Evolução mensal da efetividade</CardTitle></CardHeader>
          <CardContent>
            <ResponsiveContainer width="100%" height={280}>
              <LineChart data={evolution}>
                <CartesianGrid strokeDasharray="3 3" className="stroke-border" />
                <XAxis dataKey="month" tick={{ fill: 'hsl(var(--muted-foreground))', fontSize: 12 }} />
                <YAxis domain={[0, 100]} tick={{ fill: 'hsl(var(--muted-foreground))', fontSize: 12 }} />
                <Tooltip />
                <Line type="monotone" dataKey="media" name="Média" stroke="hsl(var(--brand))" strokeWidth={2} dot={{ r: 4 }} />
              </LineChart>
            </ResponsiveContainer>
          </CardContent>
        </Card>
        <Card>
          <CardHeader><CardTitle className="text-base">Monitorias por agente</CardTitle></CardHeader>
          <CardContent>
            <ResponsiveContainer width="100%" height={280}>
              <BarChart data={perAgent}>
                <CartesianGrid strokeDasharray="3 3" className="stroke-border" />
                <XAxis dataKey="name" tick={{ fill: 'hsl(var(--muted-foreground))', fontSize: 12 }} />
                <YAxis tick={{ fill: 'hsl(var(--muted-foreground))', fontSize: 12 }} allowDecimals={false} />
                <Tooltip />
                <Bar dataKey="total" name="Monitorias" fill="hsl(var(--chart-1))" radius={[6, 6, 0, 0]} />
              </BarChart>
            </ResponsiveContainer>
          </CardContent>
        </Card>
        <Card>
          <CardHeader><CardTitle className="text-base">Distribuição por canal</CardTitle></CardHeader>
          <CardContent>
            <ResponsiveContainer width="100%" height={280}>
              <PieChart>
                <Pie data={channelDist} dataKey="value" nameKey="name" cx="50%" cy="50%" outerRadius={90} label>
                  <Cell fill="hsl(var(--chart-1))" />
                  <Cell fill="hsl(var(--brand))" />
                </Pie>
                <Legend />
                <Tooltip />
              </PieChart>
            </ResponsiveContainer>
          </CardContent>
        </Card>
        <Card>
          <CardHeader><CardTitle className="text-base">Desempenho por critério</CardTitle></CardHeader>
          <CardContent>
            <ResponsiveContainer width="100%" height={280}>
              <BarChart data={critPerf} layout="vertical" margin={{ left: 10 }}>
                <CartesianGrid strokeDasharray="3 3" className="stroke-border" />
                <XAxis type="number" domain={[0, 10]} tick={{ fill: 'hsl(var(--muted-foreground))', fontSize: 12 }} />
                <YAxis type="category" dataKey="desc" tick={{ fill: 'hsl(var(--muted-foreground))', fontSize: 11 }} width={30} />
                <Tooltip />
                <Bar dataKey="media" name="Média" fill="hsl(var(--chart-2))" radius={[0, 6, 6, 0]} />
              </BarChart>
            </ResponsiveContainer>
          </CardContent>
        </Card>
      </div>
    </div>
  );
}
function StatCard({ icon: Icon, label, value, sub, accent }) {
  const ring = accent === 'brand' ? 'text-brand' : 'text-primary';
  return (
    <Card>
      <CardContent className="flex items-center gap-4 p-5">
        <div className={`flex h-11 w-11 items-center justify-center rounded-xl bg-muted ${ring}`}>
          <Icon className="h-5 w-5" />
        </div>
        <div className="min-w-0">
          <p className="text-xs font-medium uppercase tracking-wide text-muted-foreground">{label}</p>
          <p className="truncate text-lg font-bold">{value}</p>
          {sub && <p className="text-xs text-muted-foreground">{sub}</p>}
        </div>
      </CardContent>
    </Card>
  );
}

function Loading() {
  return (
    <div className="flex h-full items-center justify-center">
      <div className="h-8 w-8 animate-spin rounded-full border-4 border-muted border-t-primary" />
    </div>
  );
}
