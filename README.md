window.storage = {
  get: async (key) => {
    const value = localStorage.getItem(key);
    return value ? { value } : null;
  },
  set: async (key, value) => {
    localStorage.setItem(key, value);
    return { key, value };
  },
  delete: async (key) => {
    localStorage.removeItem(key);
    return { key, deleted: true };
  },
};
import { useState, useEffect } from "react";

const C={parchment:"#f5f0e8",parchmentDark:"#e8e0cc",ink:"#1a1410",inkLight:"#3d3228",inkFaded:"#6b5a48",inkGhost:"#a89880",stain:"#d4c9a8",red:"#7a2020",green:"#1a4a2a",gold:"#8a6a10"};
const inkFont="'Georgia','Times New Roman',serif";
const sansInk="'Courier New','Lucida Console',monospace";

const SAVE_KEY="willowroot_v5";
const saveGame=(st)=>{try{window.storage.set(SAVE_KEY,JSON.stringify(st));}catch(e){}};
const loadGame=async()=>{try{const r=await window.storage.get(SAVE_KEY);if(r?.value)return JSON.parse(r.value);}catch(e){}return null;};
const deleteSave=async()=>{try{await window.storage.delete(SAVE_KEY);}catch(e){}};

const HS=26,HCOLS=9,HROWS=7;
const LOC_HEXES={"E7":{c:4,r:3},"D3":{c:3,r:2},"G7":{c:6,r:4},"J7":{c:7,r:5},"A3":{c:1,r:1},"A7":{c:1,r:3},"C5":{c:2,r:4},"H7":{c:7,r:2},"D7":{c:3,r:5},"FL1":{c:5,r:1},"FL2":{c:8,r:3},"FL3":{c:0,r:5},"FL4":{c:2,r:6},"FL5":{c:6,r:2},"B4":{c:1,r:5},"B7":{c:4,r:6},"C3":{c:5,r:5},"X1":{c:0,r:2},"X2":{c:8,r:1},"X3":{c:6,r:0},"X4":{c:3,r:6},"X5":{c:7,r:4},"X6":{c:0,r:4},"X7":{c:5,r:6},"X8":{c:2,r:0},"X9":{c:8,r:6},"Y1":{c:0,r:0},"Y2":{c:1,r:6},"Y3":{c:7,r:0},"Y4":{c:8,r:4},"Y5":{c:3,r:0},"Y6":{c:4,r:5},"Y7":{c:6,r:6},"Y8":{c:2,r:2},"Y9":{c:5,r:3},"Y10":{c:4,r:1},"Y11":{c:0,r:6},"Y12":{c:7,r:1},"Y13":{c:6,r:3},"FL6":{c:1,r:4},"FL7":{c:3,r:1},"FL8":{c:8,r:5},"FL9":{c:0,r:3},"FL10":{c:5,r:2},"FL11":{c:2,r:5},"FL12":{c:7,r:6},"FL13":{c:4,r:0},"FL14":{c:8,r:2},"FL15":{c:1,r:2},"FL16":{c:3,r:4},"FL17":{c:6,r:5}};
const VH={c:4,r:3};
const hkey=(c,r)=>`${c},${r}`;
function hcenter(c,r){return{x:HS*1.75*c+36,y:HS*Math.sqrt(3)*(r+(c%2)*0.5)+36};}
function hcorners(cx,cy,r=HS){return Array.from({length:6},(_,i)=>{const a=Math.PI/180*(60*i);return`${cx+r*Math.cos(a)},${cy+r*Math.sin(a)}`;}).join(" ");}
function hneighbours(c,r){const e=c%2===0;return[{c:c-1,r:e?r-1:r},{c:c-1,r:e?r:r+1},{c,r:r-1},{c,r:r+1},{c:c+1,r:e?r-1:r},{c:c+1,r:e?r:r+1}].filter(h=>h.c>=0&&h.r>=0&&h.c<HCOLS&&h.r<HROWS);}
function hterrain(c,r){const v=(c*7+r*13)%17;if(v<3)return"water";if(v<6)return"dense";if(v<9)return"meadow";return"forest";}
const TF={fog:"#2C2C2A",water:"#d8e8f0",dense:"#c8d4b8",meadow:"#e8e4c8",forest:"#c0cca8",village:"#f5f0e8"};
const TS={fog:"#3d3228",water:"#8aa8b8",dense:"#6b7a58",meadow:"#a89860",forest:"#5a7040",village:"#1a1410"};
function initHexMap(){const rev=new Set();rev.add(hkey(VH.c,VH.r));hneighbours(VH.c,VH.r).forEach(h=>rev.add(hkey(h.c,h.r)));return{revealed:[...rev]};}
function revealAround(hexMap,locId){const pos=LOC_HEXES[locId];if(!pos)return hexMap;const s=new Set(hexMap.revealed||[]);s.add(hkey(pos.c,pos.r));hneighbours(pos.c,pos.r).forEach(h=>s.add(hkey(h.c,h.r)));return{...hexMap,revealed:[...s]};}

function HexMap({s,allLocs,onHexClick}){
  const revealed=new Set(s.hexMap?.revealed||[]);
  const svgW=HS*1.75*HCOLS+70,svgH=HS*Math.sqrt(3)*(HROWS+0.5)+60;
  const locByHex={};allLocs.forEach(loc=>{const pos=LOC_HEXES[loc.id];if(pos)locByHex[hkey(pos.c,pos.r)]=loc;});
  const hexes=[];
  for(let c=0;c<HCOLS;c++)for(let r=0;r<HROWS;r++){
    const key=hkey(c,r),isRev=revealed.has(key),isV=c===VH.c&&r===VH.r;
    const terrain=isV?"village":hterrain(c,r);
    const loc=locByHex[key],{x,y}=hcenter(c,r);
    const state=s.locationStates?.[loc?.id]||"calm";
    const visited=loc&&(s.exploredLocs||[]).includes(loc.id);
    hexes.push({key,c,r,x,y,isRev,isV,terrain,loc,state,visited});
  }
  return(
    <svg width="100%" viewBox={`0 0 ${svgW} ${svgH}`} style={{display:"block"}}>
      <defs><pattern id="hatch" width="6" height="6" patternUnits="userSpaceOnUse" patternTransform="rotate(45)"><line x1="0" y1="0" x2="0" y2="6" stroke="#3d3228" strokeWidth="0.8" opacity="0.4"/></pattern></defs>
      {hexes.map(h=>{
        const fill=h.isRev?TF[h.terrain]:TF.fog,stroke=h.isRev?TS[h.terrain]:TS.fog,corners=hcorners(h.x,h.y);
        const stateCol=h.state==="corrupted"?C.red:h.state==="disturbed"?C.gold:"none";
        return(<g key={h.key} onClick={()=>h.isRev&&h.loc&&onHexClick(h.loc)} style={{cursor:h.isRev&&h.loc?"pointer":"default"}}>
          <polygon points={corners} fill={fill} stroke={stroke} strokeWidth={h.isV?"2":"0.8"}/>
          {!h.isRev&&<polygon points={corners} fill="url(#hatch)" stroke="none"/>}
          {h.isRev&&h.state!=="calm"&&h.loc&&<polygon points={corners} fill="none" stroke={stateCol} strokeWidth="1.5" strokeDasharray="3,2"/>}
          {h.isV&&h.isRev&&<><circle cx={h.x} cy={h.y} r="7" fill={C.ink} opacity="0.9"/><text x={h.x} y={h.y+4} textAnchor="middle" fontSize="8" fill={C.parchment} fontWeight="bold" fontFamily={sansInk}>W</text></>}
          {h.isRev&&h.loc&&!h.isV&&<><circle cx={h.x} cy={h.y-2} r="5" fill={h.loc.fluff?C.stain:h.loc.danger?C.red:C.green} stroke={C.ink} strokeWidth="0.8" opacity="0.9"/><text x={h.x} y={h.y+1} textAnchor="middle" fontSize="6" fill={C.parchment} fontWeight="bold" fontFamily={sansInk}>{h.loc.fluff?"~":h.loc.danger?"!":"◎"}</text>{h.visited&&<text x={h.x} y={h.y+12} textAnchor="middle" fontSize="6" fill={C.inkFaded} fontFamily={sansInk}>{h.loc.name.slice(0,9)}</text>}</>}
          {!h.isRev&&<text x={h.x} y={h.y+4} textAnchor="middle" fontSize="9" fill="#5a4a38" opacity="0.5" fontFamily={sansInk}>?</text>}
        </g>);
      })}
      {[["W",C.ink,"Village"],["◎",C.green,"Location"],["!",C.red,"Danger"],["~",C.stain,"Fluff"]].map(([sym,col,lbl],i)=>(
        <g key={lbl} transform={`translate(${8+i*65},${svgH-18})`}>
          <circle cx="6" cy="6" r="5" fill={col} stroke={C.ink} strokeWidth="0.5" opacity="0.9"/>
          <text x="6" y="9.5" textAnchor="middle" fontSize="6" fill={C.parchment} fontWeight="bold" fontFamily={sansInk}>{sym}</text>
          <text x="14" y="10" fontSize="8" fill={C.inkFaded} fontFamily={sansInk}>{lbl}</text>
        </g>
      ))}
    </svg>
  );
}

const clamp=(v,mn,mx)=>Math.min(mx,Math.max(mn,v));
const Effects={
  food:  n=>s=>({...s,food:  clamp(s.food+n,  0,s.foodCap)}),
  wood:  n=>s=>({...s,wood:  clamp(s.wood+n,  0,s.woodCap)}),
  mats:  n=>s=>({...s,mats:  clamp(s.mats+n,  0,s.matsCap)}),
  morale:n=>s=>({...s,morale:clamp(s.morale+n,0,100)}),
  threat:n=>s=>({...s,threat:clamp(s.threat+n,0,10)}),
  compose:(...fns)=>s=>fns.reduce((a,f)=>f(a),s),
  fromData:d=>s=>{let ns=s;if(d.food)ns=Effects.food(d.food)(ns);if(d.wood)ns=Effects.wood(d.wood)(ns);if(d.mats)ns=Effects.mats(d.mats)(ns);if(d.morale)ns=Effects.morale(d.morale)(ns);if(d.threat)ns=Effects.threat(d.threat)(ns);return ns;},
};

function getSeason(t){
  if(t<=15)return{name:"Early Autumn",tg:0.8,ebc:0.18,label:"Early Autumn — days are still warm"};
  if(t<=30)return{name:"Late Autumn", tg:1.2,ebc:0.25,label:"Late Autumn — nights grow cold"};
  return      {name:"Pre-Winter",   tg:1.8,ebc:0.35,label:"Pre-Winter — frost on the roots"};
}
const FACTION_DEFAULTS={ants:0,patrol:0,shrine:0,spider:0};

const CHAINS={
  spider:{steps:[
    {title:"The Spider's Bargain",body:"She sits at the centre of her web like a queen at court. Her voice is low and formal. She knows where three caches are buried. She asks only that you remember the debt.",choices:[
      {label:"Accept the bargain",effect:s=>({...Effects.compose(Effects.food(8),Effects.mats(5))(s),factions:{...s.factions,spider:1},log:[...s.log,{t:s.turn,msg:"Spider's bargain accepted. +8 food, +5 mats.",good:true,title:"The Bargain Sealed",lore:"She did not shake paws. She simply waited until you nodded. The debt is remembered."}]})},
      {label:"Decline politely",effect:s=>({...s,factions:{...s.factions,spider:0},log:[...s.log,{t:s.turn,msg:"The spider's bargain declined.",good:true,title:"No Bargain",lore:"'Another time,' she said. But she sounded certain there would not be another time."}]})},
    ]},
    {title:"The Spider Calls In the Debt",body:"A single thread at the burrow entrance, tied in a specific knot. The spider requires payment.",choices:[
      {label:"Send food (costs 8 food)",effect:s=>({...Effects.food(-8)(s),factions:{...s.factions,spider:2},log:[...s.log,{t:s.turn,msg:"Debt paid with food. Spider allied.",good:true,title:"Debt Honoured",lore:"The food was gone by morning. A sense of watchful protection settles over the entrance."}]})},
      {label:"Send a volunteer (away 3 turns)",effect:s=>{const ok=s.mice.filter(m=>!m.injured&&!m.lost);if(!ok.length)return{...s,log:[...s.log,{t:s.turn,msg:"No mouse available for the spider.",good:false,title:"Debt Unpaid",lore:"She will remember this."}]};const c=pick(ok);return{...s,mice:s.mice.map(m=>m.id===c.id?{...m,lost:true,lostTurns:3,lostReason:"sent to the spider"}:m),factions:{...s.factions,spider:2},log:[...s.log,{t:s.turn,msg:`${c.name} sent to the spider.`,good:true,title:"A Mouse Sent",lore:`${c.name} walked into the webs without looking back. The spider always returns what she borrows.`}]};}},
      {label:"Refuse",effect:s=>({...Effects.threat(3)(s),factions:{...s.factions,spider:-2},log:[...s.log,{t:s.turn,msg:"Spider's debt refused. Threat +3.",good:false,title:"The Debt Refused",lore:"Three mice reported nightmares that week."}]})},
    ]},
    {title:"The Spider's Web — An Ally",body:"Word comes through the web-thread network: the spider intercepted a rat scouting party and turned them back.",choices:[
      {label:"Acknowledge the gift",effect:s=>({...Effects.compose(Effects.threat(-3),Effects.morale(8))(s),log:[...s.log,{t:s.turn,msg:"Spider ally acts — threat -3, morale +8.",good:true,title:"The Web Holds",lore:"Three rats turned back at the old stone. The spider considers this a professional matter."}]})},
    ]},
  ]},
  ants:{steps:[
    {title:"The Ant Envoy",body:"A single ant bears a tiny clay token — an old diplomatic signal. The colony proposes mutual defence.",choices:[
      {label:"Accept the accord",effect:s=>({...Effects.threat(-2)(s),factions:{...s.factions,ants:1},log:[...s.log,{t:s.turn,msg:"Ant accord accepted. Threat -2.",good:true,title:"The Ant Accord",lore:"That afternoon an ant tapped twice on the root wall: rat patrol, north side."}]})},
      {label:"Decline",effect:s=>({...s,factions:{...s.factions,ants:-1},log:[...s.log,{t:s.turn,msg:"Ant accord declined.",good:false,title:"Accord Rejected",lore:"The ant walked away with the dignity of a creature that will not forget."}]})},
    ]},
    {title:"The Colony Needs Aid",body:"The ant network goes quiet. A single worker, barely moving, reaches the burrow. Fungus-blight has struck their stores.",choices:[
      {label:"Send 6 food",effect:s=>({...Effects.food(-6)(s),factions:{...s.factions,ants:2},log:[...s.log,{t:s.turn,msg:"Colony aided. Ants allied.",good:true,title:"Aid Given",lore:"The colony recovered. The threat reports became more detailed than ever."}]})},
      {label:"Send 3 food",effect:s=>({...Effects.food(-3)(s),factions:{...s.factions,ants:1},log:[...s.log,{t:s.turn,msg:"Colony aided with 3 food.",good:true,title:"Partial Aid",lore:"Not enough, but something. The colony remembered."}]})},
      {label:"Send nothing",effect:s=>({...s,factions:{...s.factions,ants:-1},log:[...s.log,{t:s.turn,msg:"Colony not aided. Relations strained.",good:false,title:"No Aid",lore:"Relations become carefully formal — in ant terms, nearly hostile."}]})},
    ]},
    {title:"The Colony Marches",body:"The ground vibrates with ten thousand feet. The ant colony has mobilised on your behalf.",choices:[
      {label:"Witness and give thanks",effect:s=>({...Effects.morale(12)({...s,threat:0}),log:[...s.log,{t:s.turn,msg:"Ant army clears all threats. Threat→0, morale +12.",good:true,title:"The March of Allies",lore:"Row upon row, moving with one mind. When it was done, the garden was utterly still."}]})},
    ]},
  ]},
  shrine:{steps:[
    {title:"The Shrine Keeper Speaks",body:"An elderly mouse in plain robes. She offers a blessing — but first: 'What do you owe the dead?'",choices:[
      {label:"'We remember them.'",effect:s=>({...Effects.morale(10)(s),factions:{...s.factions,shrine:1},log:[...s.log,{t:s.turn,msg:"Shrine keeper pleased. Morale +10.",good:true,title:"An Honest Answer",lore:"She touched each mouse's forehead. Everyone felt, afterwards, slightly more certain."}]})},
      {label:"'We owe them winter stores.'",effect:s=>({...s,factions:{...s.factions,shrine:0},log:[...s.log,{t:s.turn,msg:"Shrine keeper considers.",good:true,title:"A Practical Answer",lore:"'Practical,' she said, in a tone that was not entirely a compliment."}]})},
      {label:"'Nothing. The dead are gone.'",effect:s=>({...Effects.morale(-5)(s),factions:{...s.factions,shrine:-1},log:[...s.log,{t:s.turn,msg:"Shrine keeper displeased. Morale -5.",good:false,title:"A Hard Answer",lore:"Three mice felt cold for the rest of the day."}]})},
    ]},
    {title:"The Keeper's Trial",body:"One mouse must sit alone at the boulder shrine through a full night, in silence, without sleeping.",choices:[
      {label:"Send your bravest mouse",effect:s=>{const b=s.mice.find(m=>m.trait==="brave"&&!m.injured&&!m.lost)||s.mice.find(m=>!m.injured&&!m.lost);if(!b)return{...s,log:[...s.log,{t:s.turn,msg:"No mouse available for the trial.",good:false,title:"Trial Refused",lore:"'Another season,' she said."}]};return{...Effects.compose(Effects.morale(15),Effects.threat(-2))(s),mice:s.mice.map(m=>m.id===b.id?{...m,history:[...(m.history||[]),"Survived the Keeper's Trial"]}:m),factions:{...s.factions,shrine:2},log:[...s.log,{t:s.turn,msg:`${b.name} completed the Keeper's Trial. Morale +15, threat -2.`,good:true,title:"The Trial Complete",lore:`${b.name} came back at dawn looking older. They worked with quiet certainty after that.`}]};}},
      {label:"Decline",effect:s=>({...s,factions:{...s.factions,shrine:Math.max(-2,s.factions.shrine-1)},log:[...s.log,{t:s.turn,msg:"Trial declined.",good:false,title:"Trial Declined",lore:"'Practical,' she said again, in exactly the same tone."}]})},
    ]},
    {title:"The Great Blessing",body:"She comes at dawn and speaks for a long time in a language older than mouse-tongue.",choices:[
      {label:"Receive the blessing",effect:s=>({...Effects.compose(Effects.morale(20),Effects.threat(-3),Effects.food(6))(s),log:[...s.log,{t:s.turn,msg:"Great Blessing — morale +20, threat -3, food +6.",good:true,title:"The Great Blessing",lore:"Nettle said she was not afraid of winter anymore. Nobody disagreed."}]})},
    ]},
  ]},
  settlement:{steps:[
    {title:"The Empty Village",body:"Scouts return from J7 with questions. The last journal entry: 'They found the thing under the kamenný kruh. We must go. Do not follow.'",choices:[
      {label:"Investigate the stone circle",effect:s=>({...s,chainFlags:{...s.chainFlags,settle_investigate:true},log:[...s.log,{t:s.turn,msg:"Stone circle investigation ordered.",good:true,title:"The Order Given",lore:"Three scouts volunteer without being asked."}]})},
      {label:"Leave it — winter first",effect:s=>({...s,chainFlags:{...s.chainFlags,settle_avoided:true},log:[...s.log,{t:s.turn,msg:"The circle left uninvestigated.",good:true,title:"Wisdom or Cowardice",lore:"The journal was burned so nobody would be tempted."}]})},
    ]},
    {title:"The Cracked Stone Circle",body:"The stones are cracked and partially melted. Wrapped in cloth: a lens, thumb-sized, that shows something different depending on who holds it.",choices:[
      {label:"Take the lens",effect:s=>({...Effects.mats(3)(s),chainFlags:{...s.chainFlags,lens:true},log:[...s.log,{t:s.turn,msg:"The lens recovered.",good:true,title:"The Lens",lore:"When held to the light, you see something — it changes depending on who looks."}]})},
      {label:"Leave the lens",effect:s=>({...Effects.morale(5)(s),chainFlags:{...s.chainFlags,lens_left:true},log:[...s.log,{t:s.turn,msg:"Lens left. Morale +5.",good:true,title:"Left Undisturbed",lore:"Something about leaving it there felt right."}]})},
    ]},
    {title:"What the Lens Showed",body:"The warmth was not from below — it was the lens focusing heat. The thing they found was not a monster. It was a map scratched into the stone.",choices:[
      {label:"Use the map",effect:s=>({...Effects.threat(-4)(s),foodCap:Math.min(s.foodCap+10,90),chainFlags:{...s.chainFlags,map_found:true},log:[...s.log,{t:s.turn,msg:"Ancient map decoded. Threat -4, food cap +10.",good:true,title:"The Map of Five Hexes",lore:"The previous village left in panic over something they didn't understand. Willowroot understands."}]})},
    ]},
  ]},
};

const WORLD_EVENTS=[
  {type:"good",title:"Spilled Jam Jar",      short:"+12 food.",             lore:"By morning the foragers had licked every drop and come home smelling of strawberries.",food:12,wood:0,mats:0,morale:0,threat:0},
  {type:"good",title:"Sunflower Drop",       short:"+8 food, +3 mats.",     lore:"The great yellow head came down like distant thunder. The mice stood around it before anyone thought to start pulling seeds.",food:8,wood:0,mats:3,morale:0,threat:0},
  {type:"good",title:"Ball of Yarn",         short:"+6 wood, morale +5.",   lore:"Bramble insisted it wasn't stealing. The burrow needed insulation more than anyone needed a half-finished sock.",food:0,wood:6,mats:0,morale:5,threat:0},
  {type:"good",title:"Warm Sunny Day",       short:"Morale +10.",           lore:"For one afternoon every mouse found a warm patch of light. Acorn fell asleep on a flat stone and nobody woke her.",food:0,wood:0,mats:0,morale:10,threat:0},
  {type:"good",title:"Mushroom Ring",        short:"+10 food.",             lore:"They grow in a perfect circle the elders call the Witch's Table. The circle is not safe to sleep inside.",food:10,wood:0,mats:0,morale:0,threat:0},
  {type:"good",title:"Lost Button Hoard",    short:"+6 mats, morale +3.",   lore:"A hole in the skirting board, crammed with buttons and one lens that turned the whole world golden.",food:0,wood:0,mats:6,morale:3,threat:0},
  {type:"good",title:"Cork Windfall",        short:"+5 wood, +3 mats.",     lore:"A wine cork rolled to a stop outside the burrow entrance. Dense as anything and the size of a small shed.",food:0,wood:5,mats:3,morale:0,threat:0},
  {type:"good",title:"Wandering Peddler",    short:"+4 food, +4 mats.",     lore:"A mouse with a cart packed with extraordinary things. She traded fairly and moved on before anyone could ask where she was going.",food:4,wood:0,mats:4,morale:5,threat:0},
  {type:"good",title:"Rain After Drought",   short:"+6 food, morale +6.",   lore:"The meadow greened overnight. The foragers came back so laden they could barely walk.",food:6,wood:0,mats:0,morale:6,threat:0},
  {type:"good",title:"Sleeping Cat",         short:"Threat -3, morale +4.", lore:"The great predator is asleep in the sun. The scouts moved freely for hours. Everyone knows it won't last.",food:0,wood:0,mats:0,morale:4,threat:-3},
  {type:"good",title:"Old Sock Cache",       short:"+5 wood, morale +2.",   lore:"A lost sock, filled with nesting material by some previous resident. Left as if for someone who never came.",food:0,wood:5,mats:0,morale:2,threat:0},
  {type:"good",title:"Friendly Frog",        short:"Threat -2, morale +6.", lore:"She arrived at dusk in a coat of felted moss, speaking in the old way — slow and formal, as frogs do. She traded safe-path knowledge for warmth.",food:0,wood:0,mats:0,morale:6,threat:-2},
  {type:"good",title:"Acorn Avalanche",      short:"+9 food.",              lore:"A squirrel above dislodged its winter cache. The mice below did not ask questions. They simply carried.",food:9,wood:0,mats:0,morale:3,threat:0},
  {type:"good",title:"Forgotten Matchbox",   short:"+4 wood, +4 mats.",     lore:"Wedged behind a drainpipe — enormous, full of small useful things. The striking strip alone was worth the trip.",food:0,wood:4,mats:4,morale:0,threat:0},
  {type:"good",title:"The Long Sleep",       short:"Morale +8, threat -1.", lore:"For three days the garden was quiet. Whatever hunts here took a rest. The village breathed.",food:0,wood:0,mats:0,morale:8,threat:-1},
  {type:"good",title:"Paper Nest Found",     short:"+5 mats.",              lore:"A wasp nest from last year, abandoned and dry. The papery walls pull apart into perfect insulating sheets.",food:0,wood:0,mats:5,morale:0,threat:0},
  {type:"good",title:"Fruit Fallen Early",   short:"+7 food.",              lore:"The apple came down before the birds found it. By the time they circled back, the mice had taken everything useful.",food:7,wood:0,mats:0,morale:2,threat:0},
  {type:"good",title:"The Elders Tell Stories",short:"Morale +12.",         lore:"On a quiet evening, the eldest mouse sat by the hearthstone and told stories until long past the usual sleeping hour.",food:0,wood:0,mats:0,morale:12,threat:0},
  {type:"good",title:"Grass Snake Evicts Rats",short:"Threat -3.",          lore:"She is not interested in mice. She is interested in rat-sized prey, and she has decided the eastern wall is her territory. The rats left within a day.",food:0,wood:0,mats:0,morale:3,threat:-3},
  {type:"good",title:"Hedgehog Passes Through",short:"Threat -3, morale +5.",lore:"She moved through the garden at dusk with the unhurried authority of something that has no natural enemies.",food:0,wood:0,mats:0,morale:5,threat:-3},
  {type:"good",title:"The Long Rain",        short:"+8 food, +4 wood.",     lore:"Five days of steady rain. The mushrooms came up overnight. The worms surfaced.",food:8,wood:4,mats:0,morale:0,threat:0},
  {type:"good",title:"Swallows Arrive",      short:"Threat -2, morale +7.", lore:"They came back, which means the insects are back.",food:0,wood:0,mats:0,morale:7,threat:-2},
  {type:"good",title:"Child Left Food Out",  short:"+10 food.",             lore:"A sandwich, half-eaten, left on the garden wall. The mice moved on it before the birds did.",food:10,wood:0,mats:0,morale:3,threat:0},
  {type:"good",title:"The Fox Moves On",     short:"Threat -4.",            lore:"Her scent-marks near the wall are old now. The mice simply breathe.",food:0,wood:0,mats:0,morale:6,threat:-4},
  {type:"good",title:"Good Dreams",          short:"Morale +9.",            lore:"Nobody could explain it. Three nights in a row, everyone slept deeply and woke rested.",food:0,wood:0,mats:0,morale:9,threat:0},
  {type:"bad", title:"Cat Sighting",         short:"Morale -12, threat +3.",lore:"It didn't even look their way. But no one spoke above a whisper for the rest of the day.",food:0,wood:0,mats:0,morale:-12,threat:3},
  {type:"bad", title:"Mold in the Stores",   short:"Food -8.",              lore:"Green and grey and smelling of old rain. Clover cried a little. Nobody blamed her.",food:-8,wood:0,mats:0,morale:0,threat:0},
  {type:"bad", title:"Rain Flood",           short:"Mats -5, wood -3.",     lore:"Cold dark water before dawn. They salvaged what they could by lantern-light.",food:0,wood:-3,mats:-5,morale:0,threat:0},
  {type:"bad", title:"Owl Shadow",           short:"Morale -8, threat +2.", lore:"One moment the moon was bright. The next — darkness, and a sound like vast slow wings.",food:0,wood:0,mats:0,morale:-8,threat:2},
  {type:"bad", title:"Beetle Raid",          short:"Food -6, mats -2.",     lore:"They came in the night in an orderly column that showed grim professionalism.",food:-6,wood:0,mats:-2,morale:0,threat:0},
  {type:"bad", title:"An Injury",            short:"One mouse injured.",    lore:"A small yelp, then silence. Someone fetched the acorn-cap of rosehip salve.",food:0,wood:0,mats:0,morale:-5,threat:0,special:"injure"},
  {type:"bad", title:"Rat Scouts",           short:"Threat +3, morale -5.", lore:"Seven with notched ears moving along the base of the wall. They didn't find the entrance. But they came close.",food:0,wood:0,mats:0,morale:-5,threat:3},
  {type:"bad", title:"Fungus in the Wood",   short:"Wood -5, morale -3.",   lore:"The woodpile has been invaded. White threads run through carefully stacked timber.",food:0,wood:-5,mats:0,morale:-3,threat:0},
  {type:"bad", title:"Fox at the Wall",      short:"Threat +4, morale -8.", lore:"Foxes are intelligent in a way that makes mice deeply uneasy.",food:0,wood:0,mats:0,morale:-8,threat:4},
  {type:"bad", title:"Cold Snap",            short:"Food -4, morale -5.",   lore:"Three days of cold that came too early. The foragers found little.",food:-4,wood:0,mats:0,morale:-5,threat:0},
  {type:"bad", title:"Magpie Pair",          short:"Threat +2, mats -3.",   lore:"Two of them, working together in a way that felt deliberate and unkind.",food:0,wood:0,mats:-3,morale:-4,threat:2},
  {type:"bad", title:"Early Dark",           short:"Food -3, morale -7.",   lore:"The days shortened so quickly this week that foragers twice found themselves caught outside after dark.",food:-3,wood:0,mats:0,morale:-7,threat:1},
  {type:"bad", title:"Flash Flood",          short:"Food -5, wood -4.",     lore:"The drain blocked and the lower garden became a shallow lake overnight.",food:-5,wood:-4,mats:0,morale:-4,threat:0},
  {type:"bad", title:"Crow",                 short:"Threat +3, morale -8.", lore:"One crow is more dangerous than five rats. It is smart, it remembers, and it saw the burrow entrance.",food:0,wood:0,mats:0,morale:-8,threat:3},
  {type:"bad", title:"The Argument Spreads", short:"Morale -10.",           lore:"What began between two mice spread until half the village had taken sides.",food:0,wood:0,mats:0,morale:-10,threat:0},
  {type:"bad", title:"Bad Batch",            short:"Food -7.",              lore:"The mushrooms looked right. The smell was right. But three mice spent two days very ill.",food:-7,wood:0,mats:0,morale:-4,threat:0},
  {type:"bad", title:"Humans Leave Gate Open",short:"Threat +4.",           lore:"The garden gate stood open all day. By the time anyone noticed, three separate unknown animals had investigated.",food:0,wood:0,mats:0,morale:-5,threat:4},
  {type:"bad", title:"Rot in the Wall",      short:"Wood -6.",              lore:"The section of wall the foragers relied on for shelter is worse than anyone thought.",food:0,wood:-6,mats:0,morale:-3,threat:0},
  {type:"bad", title:"Strange Tracks",       short:"Threat +3.",            lore:"Nobody recognised them. Something else walked through the garden in the night.",food:0,wood:0,mats:0,morale:-6,threat:3},
  {type:"bad", title:"Supply Line Breaks",   short:"Food -4, mats -3.",     lore:"The route between the burrow and the main foraging ground was blocked.",food:-4,wood:0,mats:-3,morale:-4,threat:0},
  {type:"bad", title:"The Sickness",         short:"One mouse injured, morale -6.",lore:"Not an injury — something quieter. A mouse spent two days unable to work.",food:0,wood:0,mats:0,morale:-6,threat:0,special:"injure"},
  {type:"bad", title:"Heron Learns the Routes",short:"Threat +3, morale -6.",lore:"It has been watching. The scouts realised when it moved to exactly where they had been planning to go.",food:0,wood:0,mats:0,morale:-6,threat:3},
];

const ALL_LOCATIONS=[
  {id:"E7",name:"Spider's Domain",danger:true,chain:"spider",fluff:false,desc:{calm:"Fine webs almost invisible. The spider is patient and intelligent.",disturbed:"The webs are denser. The spider has been expanding.",corrupted:"The webs cover everything. Something large caught in the outer rings."},safe:{calm:"Your scouts sense the webs and retreat cleanly.",disturbed:"New webs nearly caught one scout. You pull back.",corrupted:"Your scouts take one look and turn around."},outcomes:{calm:[{w:2,type:"good",title:"The Spider Speaks",lore:"She offers information in exchange for a remembered debt.",food:0,wood:0,mats:0,morale:0,threat:-2,special:"spider_chain"},{w:2,type:"good",title:"Web Silk Harvested",lore:"Abandoned sections — incredibly strong.",food:0,wood:0,mats:6,morale:0,threat:0},{w:1,type:"bad",title:"Caught in the Web",lore:"The spider watched with patient eyes until others cut her free.",food:0,wood:0,mats:0,morale:-5,threat:0,special:"injure",locState:"disturbed"}],disturbed:[{w:2,type:"bad",title:"The Webs Tighten",lore:"Your scouts barely made it out.",food:0,wood:0,mats:0,morale:0,threat:2,locState:"corrupted"},{w:1,type:"good",title:"She is Distracted",lore:"Something else has her attention. You slip through safely.",food:0,wood:0,mats:4,morale:0,threat:0}],corrupted:[{w:2,type:"bad",title:"No Entry",lore:"The territory is impassable.",food:0,wood:0,mats:0,morale:0,threat:1},{w:1,type:"good",title:"The Debt Remembered",lore:"A thread left as a guide.",food:0,wood:0,mats:3,morale:0,threat:-1}]}},
  {id:"D3",name:"Enormous Ant Colony",danger:true,chain:"ants",fluff:false,desc:{calm:"The earth pulses with life. Ants march in endless streams.",disturbed:"The streams are disrupted. Patrols more aggressive.",corrupted:"The colony on war-footing. Every entrance guarded."},safe:{calm:"Your scouts watch from a safe distance.",disturbed:"The patrols turn toward your scouts.",corrupted:"Armed workers at every approach."},outcomes:{calm:[{w:3,type:"good",title:"Pheromone Trade",lore:"A drop of fruit juice. The ants responded with food scraps.",food:9,wood:0,mats:0,morale:0,threat:0,special:"ants_chain"},{w:1,type:"bad",title:"Colony Attacked",lore:"Something went wrong. The wave mobilised.",food:0,wood:0,mats:0,morale:-8,threat:2,locState:"disturbed"}],disturbed:[{w:2,type:"good",title:"Ally Signal",lore:"The clay token recognised.",food:0,wood:0,mats:0,morale:0,threat:0,special:"ants_chain2"},{w:2,type:"bad",title:"Turned Back",lore:"Mandibles and numbers.",food:0,wood:0,mats:0,morale:0,threat:1}],corrupted:[{w:1,type:"good",title:"Old Alliance",lore:"A single ant breaks formation.",food:0,wood:0,mats:0,morale:0,threat:0,special:"ants_chain3"},{w:3,type:"bad",title:"War Footing",lore:"Caught in the edge of a war.",food:0,wood:0,mats:0,morale:-5,threat:2}]}},
  {id:"G7",name:"Mouse Shrine",danger:false,chain:"shrine",fluff:false,desc:{calm:"A small stone altar with offerings. Someone maintains it.",disturbed:"The offerings rearranged. The keeper is active.",corrupted:"The shrine is unattended. Offerings blown away."},safe:{calm:"Your scouts leave a small seed.",disturbed:"Fresh offerings — the keeper was recently here.",corrupted:"Your scouts tidy the altar as best they can."},outcomes:{calm:[{w:3,type:"good",title:"Shrine Blessing",lore:"The quiet here is different from other quiets.",food:0,wood:0,mats:0,morale:12,threat:0,special:"shrine_chain"},{w:1,type:"bad",title:"Offering Knocked Over",lore:"By accident. The atmosphere changed immediately.",food:0,wood:0,mats:0,morale:-5,threat:0}],disturbed:[{w:2,type:"good",title:"Keeper Encountered",lore:"She looked at each scout before speaking.",food:3,wood:0,mats:0,morale:8,threat:0,special:"shrine_chain2"},{w:1,type:"bad",title:"Strange Omen",lore:"The offerings arranged into a warning none could read.",food:0,wood:0,mats:0,morale:-4,threat:1}],corrupted:[{w:1,type:"good",title:"She Left a Message",lore:"One word scratched on the altar: 'Endure.'",food:0,wood:0,mats:0,morale:6,threat:0},{w:2,type:"bad",title:"Silent Stone",lore:"Nothing. Just an old stone.",food:0,wood:0,mats:0,morale:-4,threat:0}]}},
  {id:"J7",name:"Abandoned Settlement",danger:true,chain:"settlement",fluff:false,desc:{calm:"Remains of nests speak of sudden departure. Food left on tables. Fires unextinguished.",disturbed:"The settlement has been searched. The journal is gone.",corrupted:"Something else has moved in. Nests rebuilt in wrong shapes."},safe:{calm:"Your scouts walk the empty streets in silence.",disturbed:"Someone else has been here.",corrupted:"The wrong-shaped nests. Leave immediately."},outcomes:{calm:[{w:2,type:"good",title:"Full Salvage",lore:"The stores still full. Someone else's winter preparations became yours.",food:10,wood:5,mats:6,morale:0,threat:0,special:"settlement_chain"},{w:1,type:"good",title:"Survivor Found",lore:"One mouse hiding in a thimble. She joins Willowroot.",food:0,wood:0,mats:0,morale:8,threat:0,special:"add_mouse"},{w:1,type:"bad",title:"Why They Left",lore:"The scouts found out the hard way.",food:0,wood:0,mats:0,morale:-10,threat:4,locState:"disturbed"}],disturbed:[{w:1,type:"good",title:"Overlooked Cache",lore:"Whoever searched missed the floor panel.",food:0,wood:0,mats:4,morale:0,threat:0},{w:2,type:"bad",title:"Occupied",lore:"Not empty anymore.",food:0,wood:0,mats:0,morale:0,threat:3,locState:"corrupted"}],corrupted:[{w:2,type:"bad",title:"Wrong Shapes",lore:"Your scouts don't look long enough to identify the occupants.",food:0,wood:0,mats:0,morale:-7,threat:2}]}},
  {id:"A3",name:"Root Caves",danger:false,chain:null,fluff:false,desc:{calm:"Narrow tunnels beneath roots, smelling of wet earth.",disturbed:"Water levels higher. Some tunnels flooded.",corrupted:"Fully flooded."},safe:{calm:"Scouts map the outer tunnels.",disturbed:"The water line is worrying.",corrupted:"Flooded solid."},outcomes:{calm:[{w:3,type:"good",title:"Old Cache",lore:"Seeds, mushroom, thread. Left by someone who never came back.",food:8,wood:0,mats:3,morale:0,threat:0},{w:1,type:"bad",title:"Flash Flood",lore:"The water rose without warning.",food:0,wood:0,mats:0,morale:0,threat:0,special:"injure",locState:"disturbed"}],disturbed:[{w:1,type:"good",title:"High Shelf Cache",lore:"Above the waterline — whoever left this planned well.",food:4,wood:0,mats:0,morale:0,threat:0},{w:2,type:"bad",title:"Getting Worse",lore:"The tunnels narrowing with water.",food:0,wood:0,mats:0,morale:0,threat:0,locState:"corrupted"}],corrupted:[{w:1,type:"good",title:"Eventually Drains",lore:"Slightly receded.",food:3,wood:0,mats:0,morale:0,threat:0,locState:"calm"},{w:2,type:"bad",title:"Still Flooded",lore:"Come back later.",food:0,wood:0,mats:0,morale:0,threat:0}]}},
  {id:"A7",name:"Abandoned Farm Wall",danger:false,chain:null,fluff:false,desc:{calm:"Bricks riddled with mouse entrances. Inside: corridors and nests.",disturbed:"Someone new has moved into the upper chambers.",corrupted:"Fully occupied. Not by mice."},safe:{calm:"Scouts peer through entrances carefully.",disturbed:"Signs of activity above.",corrupted:"Something large lives in there."},outcomes:{calm:[{w:3,type:"good",title:"Wall Hoard",lore:"Buttons, copper wire, a needle the size of a sword.",food:0,wood:4,mats:7,morale:0,threat:0},{w:1,type:"bad",title:"Already Occupied",lore:"Not abandoned. A large something chased your scouts out.",food:0,wood:0,mats:0,morale:-8,threat:3,locState:"disturbed"}],disturbed:[{w:1,type:"good",title:"Lower Cache",lore:"The new occupants haven't found the lower chambers.",food:0,wood:0,mats:4,morale:0,threat:0},{w:2,type:"bad",title:"Contested",lore:"Two factions of squatters arguing.",food:0,wood:0,mats:0,morale:0,threat:2,locState:"corrupted"}],corrupted:[{w:2,type:"bad",title:"Impassable",lore:"Whatever is in there, it's settled.",food:0,wood:0,mats:0,morale:0,threat:0},{w:1,type:"good",title:"They Left Again",lore:"The occupant moved on.",food:0,wood:0,mats:0,morale:0,threat:0,locState:"calm"}]}},
  {id:"C5",name:"Wasp Nest",danger:true,chain:null,fluff:false,desc:{calm:"A hollow tree hosts an enormous wasp nest.",disturbed:"The nest disturbed. An angry buzzing.",corrupted:"The nest is dead. Grey and silent."},safe:{calm:"Your scouts make a wide circle.",disturbed:"The buzzing is agitated.",corrupted:"Safely passable now."},outcomes:{calm:[{w:1,type:"good",title:"Safe Passage",lore:"A sheltered path the wasps ignore.",food:0,wood:0,mats:0,morale:0,threat:-2},{w:1,type:"good",title:"Old Honeycomb",lore:"Crystallised and safe. Tasted like summer.",food:5,wood:0,mats:0,morale:10,threat:0},{w:2,type:"bad",title:"Stung",lore:"One sting per mouse is not something you walk off.",food:0,wood:0,mats:0,morale:-8,threat:0,special:"injure",locState:"disturbed"}],disturbed:[{w:2,type:"bad",title:"Very Angry",lore:"Whatever disturbed the nest made them worse.",food:0,wood:0,mats:0,morale:-5,threat:2},{w:1,type:"good",title:"Ignored",lore:"Too focused on their own crisis.",food:0,wood:0,mats:3,morale:0,threat:0}],corrupted:[{w:3,type:"good",title:"Dead Nest Salvaged",lore:"Beeswax, comb frames, propolis.",food:0,wood:0,mats:8,morale:0,threat:0,locState:"calm"}]}},
  {id:"H7",name:"Old Beehive",danger:false,chain:null,fluff:false,desc:{calm:"Abandoned hive. The honey is old but valuable.",disturbed:"Some bees have returned.",corrupted:"Fully recolonised."},safe:{calm:"Scouts inspect from a distance. No bees.",disturbed:"The buzzing is back.",corrupted:"Active hive."},outcomes:{calm:[{w:2,type:"good",title:"Old Honey",lore:"Dark as treacle, sweet as memory.",food:8,wood:0,mats:0,morale:7,threat:0},{w:2,type:"good",title:"Wax and Comb",lore:"Beeswax doesn't rot.",food:0,wood:0,mats:7,morale:0,threat:0},{w:1,type:"bad",title:"Not Abandoned",lore:"Three bees. That's all it took.",food:0,wood:0,mats:0,morale:0,threat:0,special:"injure",locState:"disturbed"}],disturbed:[{w:1,type:"good",title:"Small Harvest",lore:"Quick in-and-out.",food:3,wood:0,mats:0,morale:0,threat:0},{w:2,type:"bad",title:"More Bees",lore:"Getting worse.",food:0,wood:0,mats:0,morale:0,threat:0,locState:"corrupted"}],corrupted:[{w:1,type:"good",title:"Abandoned Again",lore:"The new colony moved on.",food:0,wood:0,mats:0,morale:0,threat:0,locState:"calm"},{w:2,type:"bad",title:"Active Colony",lore:"Definitely not going in there.",food:0,wood:0,mats:0,morale:0,threat:0}]}},
  {id:"D7",name:"Buried Cart",danger:false,chain:null,fluff:false,desc:{calm:"Half-sunken human cart. Natural chambers between the planks.",disturbed:"The cart has shifted — more sunken.",corrupted:"Mostly underground."},safe:{calm:"Scouts circle carefully.",disturbed:"Creaking under new weight.",corrupted:"Mostly inaccessible."},outcomes:{calm:[{w:3,type:"good",title:"Full Salvage",lore:"Ropes, iron nails, canvas. Four trips.",food:0,wood:4,mats:8,morale:0,threat:0},{w:1,type:"bad",title:"Rotten Floor",lore:"The wood looked solid. She landed badly.",food:0,wood:0,mats:0,morale:0,threat:0,special:"injure"}],disturbed:[{w:1,type:"good",title:"One Chamber",lore:"One room still dry.",food:0,wood:0,mats:3,morale:0,threat:0},{w:2,type:"bad",title:"Unstable",lore:"Dangerous.",food:0,wood:0,mats:0,morale:0,threat:0}],corrupted:[{w:2,type:"bad",title:"Gone",lore:"Fully submerged.",food:0,wood:0,mats:0,morale:0,threat:0}]}},
  {id:"X3",name:"The Blackberry Thicket",danger:false,chain:null,fluff:false,desc:{calm:"A dense tangle of canes. The berries, when they come, are enormous.",disturbed:"The canes have been disturbed.",corrupted:"Something large has pushed through."},safe:{calm:"The scouts identify the best approach routes.",disturbed:"Harvest signs suggest competition.",corrupted:"Trampled. The berries are gone."},outcomes:{calm:[{w:3,type:"good",title:"Berry Harvest",lore:"August sweetness. Six mice came back purple-pawed.",food:11,wood:0,mats:0,morale:6,threat:0},{w:1,type:"bad",title:"Thorns",lore:"The route in was harder than it looked.",food:2,wood:0,mats:0,morale:-3,threat:0,special:"injure"}],disturbed:[{w:2,type:"good",title:"Late Pickings",lore:"The first harvesters left plenty.",food:6,wood:0,mats:0,morale:3,threat:0},{w:1,type:"bad",title:"Competitor Warning",lore:"A small pile of bones at the entrance.",food:0,wood:0,mats:0,morale:-5,threat:2}],corrupted:[{w:2,type:"bad",title:"Trampled",lore:"Whatever went through here was large.",food:0,wood:0,mats:0,morale:-3,threat:1}]}},
  {id:"Y12",name:"The Sunflower Stand",danger:false,chain:null,fluff:false,desc:{calm:"A row of sunflowers past their peak, heads drooping with seed weight.",disturbed:"Birds have been at the seed heads.",corrupted:"The stalks have been cut by the humans."},safe:{calm:"Scouts assess the seed head weight.",disturbed:"The birds are still visiting regularly.",corrupted:"Just stalks. Too late."},outcomes:{calm:[{w:3,type:"good",title:"Seed Harvest",lore:"The scouts climbed the stalks and cut seeds loose for three hours.",food:12,wood:0,mats:0,morale:5,threat:0},{w:1,type:"bad",title:"Stalk Unstable",lore:"The head's weight had weakened the stalk.",food:5,wood:0,mats:0,morale:-3,threat:0,special:"injure"}],disturbed:[{w:2,type:"good",title:"Secondary Harvest",lore:"The birds left the lower heads. Still enough.",food:7,wood:0,mats:0,morale:2,threat:0},{w:1,type:"bad",title:"Bird Confrontation",lore:"The birds are aggressive and numerous.",food:3,wood:0,mats:0,morale:-4,threat:1}],corrupted:[{w:2,type:"bad",title:"Cleared",lore:"The humans cut them down.",food:0,wood:0,mats:0,morale:-3,threat:0}]}},
  {id:"X9",name:"The Rat King's Corner",danger:true,chain:null,fluff:false,desc:{calm:"In the far corner where two walls meet, someone has established territory.",disturbed:"The corner is more heavily marked.",corrupted:"The corner is a fortified position now."},safe:{calm:"Your scouts observe the marks from a distance.",disturbed:"The new markings are aggressive.",corrupted:"Your scouts see the fortifications and turn around immediately."},outcomes:{calm:[{w:1,type:"good",title:"Diplomatic Contact",lore:"The negotiation was tense and formal and resulted in a seasonal agreement.",food:2,wood:2,mats:2,morale:0,threat:-2},{w:2,type:"bad",title:"Patrol Encounter",lore:"Three rats, professional, unhurried.",food:0,wood:0,mats:0,morale:-8,threat:3,locState:"disturbed"},{w:1,type:"bad",title:"Tribute Demanded",lore:"A note, left at the burrow entrance. Formal, clear, outrageous.",food:-4,wood:-2,mats:0,morale:-6,threat:2}],disturbed:[{w:2,type:"bad",title:"Show of Force",lore:"Seven rats in formation.",food:0,wood:0,mats:0,morale:-10,threat:3,locState:"corrupted"},{w:1,type:"good",title:"Internal Dispute",lore:"The rats were arguing. In the confusion, your scouts retrieved a small cache.",food:0,wood:0,mats:5,morale:3,threat:0}],corrupted:[{w:3,type:"bad",title:"Fortress",lore:"Double perimeter, rotating guards.",food:0,wood:0,mats:0,morale:-6,threat:4}]}},
  {id:"FL1",name:"The Cracked Flowerpot",danger:false,chain:null,fluff:true,desc:{calm:"A terracotta pot lies on its side, split clean through. Someone scratched a tiny map into the inner wall.",disturbed:"The pot has been moved slightly.",corrupted:"The pot is gone. Only the indentation remains."},safe:{calm:"The scouts sit inside the cool shade for a while.",disturbed:"Something moved this.",corrupted:"Nothing left to see."},outcomes:{calm:[{w:3,type:"fluff",title:"A Quiet Lunch",lore:"The scouts ate their seeds inside the cool curve of the pot and watched a beetle walk past.",food:0,wood:0,mats:0,morale:3,threat:0},{w:1,type:"fluff",title:"The Map on the Wall",lore:"One scout spent a long time tracing the scratched lines. She made a copy.",food:0,wood:0,mats:0,morale:2,threat:0}],disturbed:[{w:2,type:"fluff",title:"Someone Was Here",lore:"Fresh soil disturbance. The scouts look around, find nothing, and go home.",food:0,wood:0,mats:0,morale:0,threat:0}],corrupted:[{w:1,type:"fluff",title:"Just the Indentation",lore:"A pot-shaped shadow in the dry earth.",food:0,wood:0,mats:0,morale:0,threat:0}]}},
  {id:"FL5",name:"The Singing Wire",danger:false,chain:null,fluff:true,desc:{calm:"A length of wire vibrates in the wind and produces a faint continuous tone.",disturbed:"The wire has gone slack. It no longer sings.",corrupted:"The wire is gone."},safe:{calm:"Your scouts sit near the wire for a while, listening.",disturbed:"The silence where the singing was feels like something missing.",corrupted:"Just an empty stretch of garden."},outcomes:{calm:[{w:3,type:"fluff",title:"The Wire's Song",lore:"Three scouts sat by the wire for an afternoon. They came home having composed a small melody.",food:0,wood:0,mats:0,morale:8,threat:0},{w:1,type:"fluff",title:"The Frequency",lore:"The Clever mouse discovered touching the wire at exactly the right point made beetles stop moving.",food:0,wood:0,mats:0,morale:3,threat:-1}],disturbed:[{w:2,type:"fluff",title:"Gone Slack",lore:"The silence is worse than the sound was.",food:0,wood:0,mats:0,morale:-2,threat:0}],corrupted:[{w:1,type:"fluff",title:"Taken",lore:"Whatever song it knew, it has taken with it.",food:0,wood:0,mats:0,morale:-3,threat:0}]}},
  {id:"FL14",name:"The Old Bell",danger:false,chain:null,fluff:true,desc:{calm:"A small brass bell hanging from a hook on the fence. The wind rings it occasionally.",disturbed:"The bell has been taken off its hook and left on the ground.",corrupted:"The bell is gone."},safe:{calm:"Your scouts listen for the bell.",disturbed:"The bell lies in the grass.",corrupted:"The fence hook is empty."},outcomes:{calm:[{w:3,type:"fluff",title:"The Wind's Note",lore:"It rang while the scouts were nearby. One clear note, held longer than physics seemed to allow.",food:0,wood:0,mats:0,morale:7,threat:0},{w:1,type:"fluff",title:"Struck Deliberately",lore:"One scout struck the bell herself, quietly. Everything paused for a full minute.",food:0,wood:0,mats:0,morale:5,threat:0}],disturbed:[{w:2,type:"fluff",title:"Grounded",lore:"The bell rings against the grass now. A duller note.",food:0,wood:0,mats:0,morale:3,threat:0}],corrupted:[{w:1,type:"fluff",title:"The Empty Hook",lore:"The wind moves across the empty hook without sound.",food:0,wood:0,mats:0,morale:-3,threat:0}]}},
];

const ALL_BUILDINGS=[
  {id:"granary",    name:"Root Cellar",      icon:"▣",cost:{wood:6,mats:4}, desc:"Food cap +30",                      flavor:"A carved mouse door and a tallow candle inside.",        effect_type:"food_cap",      effect_value:30, built:false},
  {id:"workshop",   name:"Twig Workshop",    icon:"⚒",cost:{wood:8,mats:6}, desc:"Unlocks the Craft action",          flavor:"Sawdust and the smell of pine resin.",                   effect_type:"unlock_craft",   effect_value:1,  built:false},
  {id:"thornwall",  name:"Thorn Wall",       icon:"⋈",cost:{wood:4,mats:8}, desc:"Threat -1 each turn",               flavor:"Hawthorn and briar, woven tight.",                       effect_type:"threat_passive", effect_value:1,  built:false},
  {id:"seedlib",    name:"Seed Library",     icon:"▤",cost:{wood:6,mats:6}, desc:"Foragers +1 food each",             flavor:"Tiny shelves, every seed labelled in careful script.",   effect_type:"forage_bonus",   effect_value:1,  built:false},
  {id:"hearthstone",name:"Hearthstone",      icon:"△",cost:{wood:6,mats:4}, desc:"Morale never drops below 20",       flavor:"A flat warm stone at the burrow's heart.",              effect_type:"morale_floor",   effect_value:20, built:false},
  {id:"watchpost",  name:"Crow's Watch",     icon:"◉",cost:{wood:8,mats:5}, desc:"Watch action gives -3 threat",      flavor:"A cork platform in the garden fork, rope-ladder down.",  effect_type:"watch_bonus",    effect_value:1.5,built:false},
  {id:"dryroom",    name:"Drying Room",      icon:"≋",cost:{wood:5,mats:7}, desc:"Food spoilage -1 per turn",         flavor:"Herbs hanging from thread, the air always slightly warm.",effect_type:"food_preserve",  effect_value:1,  built:false},
  {id:"burrowinn",  name:"Traveller's Nook", icon:"⌂",cost:{wood:7,mats:5}, desc:"Arrival chance +20%",               flavor:"A small alcove with a fresh-woven mat and a candle stub.",effect_type:"arrival_bonus",  effect_value:0.2,built:false},
  {id:"runepath",   name:"Rune Path",        icon:"ᚱ",cost:{wood:4,mats:9}, desc:"Explore outcomes +10% favourable",  flavor:"Stones scratched with old signs, laid from the burrow entrance outward.",effect_type:"explore_luck",effect_value:0.1,built:false},
  {id:"icebox",     name:"Winter Cache",     icon:"❅",cost:{wood:9,mats:6}, desc:"Wood cap +20",                      flavor:"A deep cool chamber lined with bark and moss.",          effect_type:"wood_cap",       effect_value:20, built:false},
  {id:"watchtower", name:"Acorn Tower",      icon:"⊕",cost:{wood:10,mats:7},desc:"Threat passive -1, explore bonus",  flavor:"Three acorn caps stacked on a twig scaffold.",effect_type:"tower",effect_value:1,built:false},
  {id:"smokehouse", name:"Smokehouse",       icon:"☁",cost:{wood:7,mats:8}, desc:"Food cap +15, food spoilage -1",    flavor:"A hollowed cork with a tiny chimney of bent wire.",       effect_type:"smokehouse",     effect_value:1,  built:false},
  {id:"messroom",   name:"Common Room",      icon:"◫",cost:{wood:6,mats:6}, desc:"Rest gives +1 extra morale per mouse",flavor:"Low ceiling, acorn-cap lanterns, the smell of shared suppers.",effect_type:"rest_bonus",effect_value:1,built:false},
  {id:"forgehouse", name:"Forge Stone",      icon:"◈",cost:{wood:5,mats:10},desc:"Craft converts at better rate",     flavor:"A flat hearthstone and improvised bellows.",effect_type:"craft_bonus",effect_value:1,built:false},
  {id:"maproom",    name:"Chart Room",       icon:"⊞",cost:{wood:8,mats:8}, desc:"Hex map reveals 2 extra hexes per exploration",flavor:"Bark charts pinned to a cork wall.",effect_type:"map_bonus",effect_value:2,built:false},
];

const MOUSE_NAMES=["Pippin","Clover","Bramble","Nettle","Sedge","Acorn","Fern","Thistle","Moss","Hazel","Reed","Sorrel","Burr","Wren","Cricket","Cobble","Ember","Ash","Flint","Briar","Yarrow","Juniper","Cobnut","Thistlewick","Rushmore","Pebble","Gorse","Dew","Flax","Chestnut","Barley","Mallow","Burnet","Cress","Elder"];
const TRAITS=[
  {id:"brave",   label:"Brave",       glyph:"⚔",desc:"Reduces threat when exploring."},
  {id:"green",   label:"Green Paw",   glyph:"☘",desc:"Foragers bring +1 food/turn."},
  {id:"stocky",  label:"Stocky",      glyph:"⚒",desc:"Haulers bring +1 wood/turn."},
  {id:"clever",  label:"Clever",      glyph:"✦",desc:"Builds faster."},
  {id:"nervous", label:"Nervous",     glyph:"~",desc:"Exploration penalty."},
  {id:"cheerful",label:"Cheerful",    glyph:"♪",desc:"Morale +0.5 passively each turn."},
  {id:"greedy",  label:"Greedy",      glyph:"$",desc:"Consumes extra food."},
  {id:"careful", label:"Careful",     glyph:"◎",desc:"Steady builder."},
  {id:"swift",   label:"Swift",       glyph:"→",desc:"Exploration threat reduction doubled."},
  {id:"forager", label:"Born Forager",glyph:"⁂",desc:"Forage yields +1.5 food/turn."},
];
const ACTIONS=[
  {id:"forage", label:"Forage",      glyph:"⁂",desc:"Gather food. Green Paw and Born Forager excel."},
  {id:"haul",   label:"Haul Wood",   glyph:"⊞",desc:"Gather wood. Stocky mice carry more."},
  {id:"gather", label:"Gather Mats", glyph:"◈",desc:"Scavenge materials from the garden edges."},
  {id:"build",  label:"Build",       glyph:"⌂",desc:"Work on queued building."},
  {id:"explore",label:"Explore",     glyph:"◎",desc:"Scout the world. Reveals the hex map."},
  {id:"rest",   label:"Rest",        glyph:"☽",desc:"Recover morale and heal injuries."},
  {id:"watch",  label:"Night Watch", glyph:"◉",desc:"Reduce threat."},
  {id:"craft",  label:"Craft",       glyph:"⚒",desc:"Requires Twig Workshop. Converts mats to food and wood."},
];
const POLICIES=[
  {id:"harvest_fest", name:"Harvest Festival",    pos:"Morale +15, foragers +1",         neg:"Costs 5 food",              flavor:"The whole village dances around the acorn pile until dawn."},
  {id:"strict_ration",name:"Strict Rationing",    pos:"Food use -2/turn",                neg:"Morale -10",                flavor:"Half a seed at supper. Everyone counts the days."},
  {id:"forager_guild",name:"Forager's Guild",     pos:"Foragers +1 output",              neg:"Builders -1",               flavor:"A red acorn cap marks those who go beyond the roots."},
  {id:"night_watch",  name:"Night Watch Decree",  pos:"Threat -2/turn",                  neg:"One mouse always on watch", flavor:"Two small eyes in the dark, watching the garden wall."},
  {id:"open_burrow",  name:"Open Burrow",         pos:"Morale +10, more arrivals",       neg:"Threat +1/turn",            flavor:"Leave the door unlatched. There are others out there."},
  {id:"deep_roots",   name:"Deep Roots",          pos:"Mats +1/turn, storage cap +10",   neg:"No new buildings",          flavor:"Dig deeper before you build higher."},
  {id:"communal",     name:"Communal Larder",     pos:"Food use -1/turn, morale +5",     neg:"Greedy mice penalised",     flavor:"Elder Moss chalks a tally mark for every seed."},
  {id:"scouts",       name:"Brave Scouts",        pos:"Explore threat bonus doubled",    neg:"Scouts risk 10% injury",    flavor:"They go alone, with only a thimble-cap and a brave heart."},
  {id:"stone_law",    name:"Stone Law",           pos:"No mouse works injured",          neg:"Morale -5 on any injury",   flavor:"Three scratches on the stone: no mouse works wounded."},
  {id:"harvest_moon", name:"Harvest Moon Vigil",  pos:"Morale +8 when a mouse returns",  neg:"Threat +0.5/turn",         flavor:"Every return is celebrated loudly, which draws attention."},
  {id:"lean_season",  name:"Lean Season Accord",  pos:"Food use -1/turn, mats +1/turn",  neg:"No forager guild benefit",  flavor:"Everyone tightens their belt and puts their paws to more practical work."},
  {id:"wardens",      name:"Garden Wardens",      pos:"Threat -1/turn, explore reveals extra hex", neg:"2 mice always on Watch or Explore", flavor:"We do not wait to be found."},
];

function pick(arr){return arr[Math.floor(Math.random()*arr.length)];}
function mkMouse(n){return{id:Math.random().toString(36).slice(2),name:n||pick(MOUSE_NAMES),trait:pick(TRAITS).id,injured:false,lost:false,lostTurns:0,lostReason:"",history:[]};}
function traitBonus(trait,action){
  if(action==="forage"&&trait==="green")   return 1;
  if(action==="forage"&&trait==="forager") return 1.5;
  if(action==="forage"&&trait==="greedy")  return -0.5;
  if(action==="explore"&&trait==="brave")  return 1;
  if(action==="explore"&&trait==="swift")  return 2;
  if(action==="explore"&&trait==="nervous")return -1;
  if(action==="haul"&&trait==="stocky")    return 1;
  return 0;
}
function injureRandom(s){
  const ok=s.mice.filter(m=>!m.injured&&!m.lost);if(!ok.length)return s;
  const t=pick(ok);
  return{...s,mice:s.mice.map(m=>m.id===t.id?{...m,injured:true,history:[...(m.history||[]),"Suffered an injury"]}:m),morale:Math.max(0,s.morale-5)};
}
function pickWeighted(outcomes,s){
  const res=outcomes.map(o=>({...o,wv:typeof o.w==="function"?o.w(s):o.w}));
  const tot=res.reduce((a,o)=>a+o.wv,0);if(tot<=0)return res[0];
  let r=Math.random()*tot;for(const o of res){r-=o.wv;if(r<=0)return o;}
  return res[res.length-1];
}
function hasBldg(s,id){return s.buildings.find(b=>b.id===id)?.built;}
function applyOutcome(s,outcome,locId){
  let ns=Effects.fromData(outcome)(s);
  if(outcome.special==="injure")ns=injureRandom(ns);
  if(outcome.special==="add_mouse"&&s.mice.length<8){const nm=mkMouse();ns={...ns,mice:[...ns.mice,nm]};}
  if(outcome.special==="spider_chain") ns={...ns,chainProgress:{...ns.chainProgress,spider:Math.max(ns.chainProgress.spider||0,1)},factions:{...ns.factions,spider:Math.min(2,ns.factions.spider+1)}};
  if(outcome.special==="ants_chain")   ns={...ns,chainProgress:{...ns.chainProgress,ants:Math.max(ns.chainProgress.ants||0,1)},factions:{...ns.factions,ants:Math.min(2,ns.factions.ants+1)}};
  if(outcome.special==="ants_chain2")  ns={...ns,chainProgress:{...ns.chainProgress,ants:Math.max(ns.chainProgress.ants||0,2)}};
  if(outcome.special==="ants_chain3")  ns={...ns,chainProgress:{...ns.chainProgress,ants:Math.max(ns.chainProgress.ants||0,3)}};
  if(outcome.special==="shrine_chain") ns={...ns,chainProgress:{...ns.chainProgress,shrine:Math.max(ns.chainProgress.shrine||0,1)},factions:{...ns.factions,shrine:Math.min(2,ns.factions.shrine+1)}};
  if(outcome.special==="shrine_chain2")ns={...ns,chainProgress:{...ns.chainProgress,shrine:Math.max(ns.chainProgress.shrine||0,2)}};
  if(outcome.special==="settlement_chain")ns={...ns,chainProgress:{...ns.chainProgress,settlement:Math.max(ns.chainProgress.settlement||0,1)}};
  if(outcome.locState&&locId)ns={...ns,locationStates:{...ns.locationStates,[locId]:outcome.locState}};
  return ns;
}

function initState(){
  const mice=[mkMouse("Pippin"),mkMouse("Clover"),mkMouse("Bramble"),mkMouse("Nettle")];
  mice[0].trait="brave";mice[1].trait="green";mice[2].trait="stocky";mice[3].trait="cheerful";
  return{
    turn:1,maxTurns:50,phase:"assign",mice,assignments:{},
    food:20,foodCap:40,wood:10,woodCap:30,mats:6,matsCap:25,morale:60,threat:0,
    buildings:ALL_BUILDINGS.map(b=>({...b})),
    policies:[],buildQueue:null,
    pendingEvent:null,pendingExplore:null,pendingChain:null,policyChoices:[],
    factions:{...FACTION_DEFAULTS},
    chainProgress:{spider:0,ants:0,shrine:0,settlement:0},
    chainFlags:{},locationStates:{},
    exploredLocs:[],hexMap:initHexMap(),
    log:[{t:0,msg:"Willowroot stirs — the first cold wind rustles the oak.",good:true,title:"Willowroot Awakens",lore:"The burrow smells of old wood and damp earth. Four mice sit in the dim light of a tallow candle. Outside, the world is enormous and does not care."}],
  };
}

function checkChainTriggers(s){
  if(s.chainProgress.spider>=1&&s.factions.spider>=1&&!s.chainFlags.spider_debt&&s.turn>15) return{...s,chainFlags:{...s.chainFlags,spider_debt:true},pendingChain:{chain:"spider",step:1},phase:"chain"};
  if(s.factions.spider===2&&!s.chainFlags.spider_ally&&s.turn>25) return{...s,chainFlags:{...s.chainFlags,spider_ally:true},pendingChain:{chain:"spider",step:2},phase:"chain"};
  if(s.chainProgress.ants>=1&&!s.chainFlags.ant_first) return{...s,chainFlags:{...s.chainFlags,ant_first:true},pendingChain:{chain:"ants",step:0},phase:"chain"};
  if(s.chainProgress.ants>=2&&!s.chainFlags.ant_crisis) return{...s,chainFlags:{...s.chainFlags,ant_crisis:true},pendingChain:{chain:"ants",step:1},phase:"chain"};
  if(s.chainProgress.ants>=3&&s.factions.ants===2&&!s.chainFlags.ant_army&&s.turn>35) return{...s,chainFlags:{...s.chainFlags,ant_army:true},pendingChain:{chain:"ants",step:2},phase:"chain"};
  if(s.chainProgress.shrine>=1&&s.factions.shrine>=1&&!s.chainFlags.shrine_first) return{...s,chainFlags:{...s.chainFlags,shrine_first:true},pendingChain:{chain:"shrine",step:0},phase:"chain"};
  if(s.chainProgress.shrine>=2&&!s.chainFlags.shrine_trial) return{...s,chainFlags:{...s.chainFlags,shrine_trial:true},pendingChain:{chain:"shrine",step:1},phase:"chain"};
  if(s.chainProgress.shrine>=2&&s.factions.shrine>=2&&!s.chainFlags.shrine_blessing&&s.turn>30) return{...s,chainFlags:{...s.chainFlags,shrine_blessing:true},pendingChain:{chain:"shrine",step:2},phase:"chain"};
  if(s.chainProgress.settlement>=1&&!s.chainFlags.settle_discovery) return{...s,chainFlags:{...s.chainFlags,settle_discovery:true},pendingChain:{chain:"settlement",step:0},phase:"chain"};
  if(s.chainFlags.settle_investigate&&!s.chainFlags.settle_circle) return{...s,chainFlags:{...s.chainFlags,settle_circle:true},pendingChain:{chain:"settlement",step:1},phase:"chain"};
  if(s.chainFlags.lens&&!s.chainFlags.settle_return) return{...s,chainFlags:{...s.chainFlags,settle_return:true},pendingChain:{chain:"settlement",step:2},phase:"chain"};
  return null;
}

function checkNextPhase(ns){
  const ch=checkChainTriggers(ns);if(ch)return ch;
  const season=getSeason(ns.turn);
  const roll=Math.random();
  if(roll<(0.18+season.ebc)){
    const pool=roll<0.18?WORLD_EVENTS.filter(e=>e.type==="good"):WORLD_EVENTS.filter(e=>e.type==="bad");
    ns.pendingEvent=pick(pool);ns.phase="event";
  } else if(ns.turn%10===1&&ns.turn>1){
    ns.policyChoices=POLICIES.filter(p=>!ns.policies.includes(p.id)).sort(()=>Math.random()-0.5).slice(0,3);
    ns.phase="policy";
  } else if(ns.turn>ns.maxTurns)ns.phase="gameover";
  else ns.phase="assign";
  return ns;
}

function processTurn(s){
  let ns={...s,assignments:{}};
  const a=s.assignments,p=s.policies,season=getSeason(ns.turn);
  const counts={};ACTIONS.forEach(x=>{counts[x.id]=0;});
  s.mice.forEach(m=>{if(a[m.id]&&!m.lost)counts[a[m.id]]=(counts[a[m.id]]||0)+1;});

  const fc=s.mice.filter(m=>a[m.id]==="forage"&&!m.lost).length;
  let fb=2.5+(hasBldg(ns,"seedlib")?1:0)+(p.includes("forager_guild")?1:0);
  if(fc>=4)fb=Math.max(1,fb-0.5*(fc-3));
  s.mice.forEach(m=>{if(a[m.id]==="forage"&&!m.lost)ns.food=clamp(ns.food+fb+traitBonus(m.trait,"forage"),0,ns.foodCap);});
  s.mice.forEach(m=>{if(a[m.id]==="haul"  &&!m.lost)ns.wood=clamp(ns.wood+2+traitBonus(m.trait,"haul"),0,ns.woodCap);});
  s.mice.forEach(m=>{if(a[m.id]==="gather"&&!m.lost)ns.mats=clamp(ns.mats+2+(p.includes("deep_roots")?1:0)+(p.includes("lean_season")?1:0),0,ns.matsCap);});

  if(ns.buildQueue&&counts["build"]>0&&!p.includes("deep_roots")){
    const bldg=ns.buildings.find(b=>b.id===ns.buildQueue);
    if(bldg&&!bldg.built&&ns.wood>=bldg.cost.wood&&ns.mats>=bldg.cost.mats){
      ns.wood-=bldg.cost.wood;ns.mats-=bldg.cost.mats;
      ns.buildings=ns.buildings.map(b=>b.id===ns.buildQueue?{...b,built:true}:b);
      if(bldg.effect_type==="food_cap")ns.foodCap+=bldg.effect_value;
      if(bldg.effect_type==="wood_cap")ns.woodCap+=bldg.effect_value;
      if(bldg.effect_type==="smokehouse")ns.foodCap+=15;
      ns.log=[...ns.log,{t:ns.turn,msg:`${bldg.name} is complete!`,good:true,title:`${bldg.name} Complete`,lore:bldg.flavor||"They stood back and looked at what they'd built. It was theirs."}];
      ns.buildQueue=null;
    }
  }

  const explorers=s.mice.filter(m=>a[m.id]==="explore"&&!m.lost);
  if(explorers.length>0){
    const braveCount=explorers.filter(m=>m.trait==="brave"||m.trait==="swift").length;
    ns.threat=Math.max(0,ns.threat-braveCount*(p.includes("scouts")?2:1));
    if(hasBldg(ns,"watchtower"))ns.threat=Math.max(0,ns.threat-1);
    const loc=pick(ALL_LOCATIONS);
    const locState=ns.locationStates[loc.id]||"calm";
    ns.pendingExplore={loc,state:locState};
  }

  ns.mice=ns.mice.map(m=>{
    if(a[m.id]==="rest"&&m.injured)return{...m,injured:false};
    if(m.lost){const rem=m.lostTurns-(counts["rest"]>0?2:1);if(rem<=0)return{...m,lost:false,lostTurns:0,lostReason:"",history:[...(m.history||[]),`Returned from: ${m.lostReason}`]};return{...m,lostTurns:Math.max(0,rem)};}
    return m;
  });
  const returned=ns.mice.filter(m=>!m.lost&&s.mice.find(sm=>sm.id===m.id)?.lost);
  returned.forEach(m=>{const b=p.includes("harvest_moon")?8:5;ns.morale=clamp(ns.morale+b,0,100);ns.log=[...ns.log,{t:ns.turn,msg:`${m.name} returned home. Morale +${b}.`,good:true,title:`${m.name} Returns`,lore:"They came back changed in small ways that are hard to name. But they came back."}];});

  const restBonus=hasBldg(ns,"messroom")?1:0;
  if(counts["rest"]>0)ns.morale=clamp(ns.morale+counts["rest"]*(4+restBonus),0,100);
  const wb=hasBldg(ns,"watchpost")?3:1.5;
  if(counts["watch"]>0)ns.threat=Math.max(0,ns.threat-counts["watch"]*wb);
  if(counts["craft"]>0&&hasBldg(ns,"workshop")){
    const craftBonus=hasBldg(ns,"forgehouse");
    const u=Math.min(ns.mats,counts["craft"]*2);ns.mats-=u;
    const yld=craftBonus?Math.ceil(u*0.67):Math.floor(u/2);
    ns.wood=clamp(ns.wood+yld,0,ns.woodCap);ns.food=clamp(ns.food+yld,0,ns.foodCap);
  }

  const active=ns.mice.filter(m=>!m.lost).length;
  const eat=active-(p.includes("strict_ration")?2:0)-(p.includes("communal")?1:0)-(p.includes("lean_season")?1:0);
  const dry=(hasBldg(ns,"dryroom")?1:0)+(hasBldg(ns,"smokehouse")?1:0);
  ns.food=clamp(ns.food-eat+dry,0,ns.foodCap);
  s.mice.forEach(m=>{if(m.trait==="cheerful"&&!m.lost)ns.morale=clamp(ns.morale+0.5,0,100);});
  s.mice.forEach(m=>{if(m.trait==="greedy"  &&!m.lost)ns.food=clamp(ns.food-0.5*(p.includes("communal")?2:1),0,ns.foodCap);});
  if(active>=6){ns.morale=clamp(ns.morale-(active-5),0,100);ns.food=clamp(ns.food-0.5*(active-5),0,ns.foodCap);}
  if(ns.food<=0){ns.morale=clamp(ns.morale-8,0,100);ns.log=[...ns.log,{t:ns.turn,msg:"Empty stores — everyone goes hungry.",good:false,title:"Hungry Night",lore:"Supper was thin broth and silence."}];}
  if(hasBldg(ns,"hearthstone"))ns.morale=Math.max(20,ns.morale);
  if(hasBldg(ns,"thornwall"))ns.threat=Math.max(0,ns.threat-1);
  if(hasBldg(ns,"watchtower"))ns.threat=Math.max(0,ns.threat-1);
  if(ns.factions.ants>=1)ns.threat=Math.max(0,ns.threat-1);
  if(ns.factions.patrol>=1)ns.threat=Math.max(0,ns.threat-0.5);
  const tw=p.includes("night_watch")?2:0,op=p.includes("open_burrow")?1:0,hm=p.includes("harvest_moon")?0.5:0,wd=p.includes("wardens")?-1:0;
  ns.threat=clamp(ns.threat+season.tg+op-tw+hm+wd,0,10);
  if(ns.threat>=7)ns.morale=clamp(ns.morale-5,0,100);

  const arrBonus=hasBldg(ns,"burrowinn")?0.2:0;
  if(ns.turn%5===0&&Math.random()<(p.includes("open_burrow")?0.7:0.5)+arrBonus&&ns.mice.length<8){
    const nm=mkMouse();const tObj=TRAITS.find(t=>t.id===nm.trait)||{label:"Unknown"};
    ns.mice=[...ns.mice,nm];ns.morale=clamp(ns.morale+5,0,100);
    ns.log=[...ns.log,{t:ns.turn,msg:`${nm.name} the ${tObj.label} joined Willowroot!`,good:true,title:`${nm.name} Arrives`,lore:"They arrived at dusk with nothing but a worn satchel and a cautious smile."}];
  }
  ns.turn=ns.turn+1;
  if(ns.pendingExplore){ns.phase="explore";return ns;}
  return checkNextPhase(ns);
}

function applyPolicyImmediate(s,pol){
  if(pol.id==="harvest_fest")return Effects.compose(Effects.morale(15),Effects.food(-5))(s);
  if(pol.id==="strict_ration")return Effects.morale(-10)(s);
  if(pol.id==="communal")return Effects.morale(5)(s);
  return s;
}

// ── UI COMPONENTS ─────────────────────────────────────────────────────────────
// Larger base sizes, bolder borders
function InkBox({children,style={},fill=C.parchment}){
  return(<div style={{background:fill,border:`2.5px solid ${C.ink}`,boxShadow:`3px 3px 0 ${C.ink}`,padding:"12px 16px",position:"relative",...style}}>{children}</div>);
}
function InkBtn({children,onClick,disabled,active,style={}}){
  return(<button onClick={onClick} disabled={disabled} style={{fontFamily:sansInk,fontSize:14,fontWeight:"bold",letterSpacing:"0.04em",background:active?C.ink:C.parchment,color:active?C.parchment:C.ink,border:`2px solid ${C.ink}`,padding:"8px 14px",cursor:disabled?"not-allowed":"pointer",opacity:disabled?0.4:1,boxShadow:active?`inset 1px 1px 0 ${C.inkLight}`:`2px 2px 0 ${C.ink}`,...style}}>{children}</button>);
}
function Title({children,size=18,style={}}){
  return <div style={{fontFamily:inkFont,fontSize:size,fontStyle:"italic",color:C.ink,fontWeight:"bold",lineHeight:1.2,...style}}>{children}</div>;
}
function Label({children,style={}}){
  return <div style={{fontFamily:sansInk,fontSize:13,fontWeight:"bold",color:C.inkFaded,...style}}>{children}</div>;
}
function Body({children,style={}}){
  return <div style={{fontFamily:inkFont,fontSize:15,fontStyle:"italic",color:C.inkLight,lineHeight:1.85,...style}}>{children}</div>;
}

function MouseSVG({injured,lost}){
  return(<svg width="38" height="38" viewBox="0 0 34 34"><ellipse cx="17" cy="21" rx="11" ry="8" fill={C.parchment} stroke={C.ink} strokeWidth="1.5"/><circle cx="17" cy="13" r="7" fill={C.parchment} stroke={C.ink} strokeWidth="1.5"/><ellipse cx="10" cy="8" rx="3.5" ry="6" fill={C.parchment} stroke={C.ink} strokeWidth="1.2" transform="rotate(-18 10 8)"/><ellipse cx="24" cy="8" rx="3.5" ry="6" fill={C.parchment} stroke={C.ink} strokeWidth="1.2" transform="rotate(18 24 8)"/><circle cx="14.5" cy="13" r="1.2" fill={C.ink}/><circle cx="19.5" cy="13" r="1.2" fill={C.ink}/><path d="M14.5 17 Q17 19 19.5 17" fill="none" stroke={C.ink} strokeWidth="1.2"/>{injured&&<path d="M5 5 L29 29 M29 5 L5 29" stroke={C.red} strokeWidth="1.5" opacity="0.45"/>}{lost&&<path d="M17 5 L17 29 M5 17 L29 17" stroke={C.gold} strokeWidth="1.5" opacity="0.55"/>}</svg>);
}

function getActionYield(act,mouse,s){
  const p=s.policies;
  const fc=s.mice.filter(m=>s.assignments[m.id]==="forage"&&!m.lost).length;
  if(act.id==="forage"){let base=2.5+(hasBldg(s,"seedlib")?1:0)+(p.includes("forager_guild")?1:0);if(fc>=4)base=Math.max(1,base-0.5*(fc-3));const b=traitBonus(mouse.trait,"forage");const parts=[`+${(base+b).toFixed(1)} food`];if(b>0)parts.push(`(+${b} ${TRAITS.find(t=>t.id===mouse.trait)?.label})`);if(b<0)parts.push(`(${b})`);return parts.join(" ");}
  if(act.id==="haul"){const b=traitBonus(mouse.trait,"haul");return`+${2+b} wood`+(b>0?` (+${b})`:"");}
  if(act.id==="gather")return`+${2+(p.includes("deep_roots")?1:0)+(p.includes("lean_season")?1:0)} mats`;
  if(act.id==="rest")  return mouse.injured?"heals injury":`+${4+(hasBldg(s,"messroom")?1:0)} morale`;
  if(act.id==="watch"){return`−${hasBldg(s,"watchpost")?3:1.5} threat`;}
  if(act.id==="craft") return hasBldg(s,"workshop")?(hasBldg(s,"forgehouse")?"mats→food+wood (bonus)":"mats→food+wood"):"needs workshop";
  if(act.id==="build") return s.buildQueue?`→ ${s.buildings.find(b=>b.id===s.buildQueue)?.name||"?"}` :"queue a building first";
  if(act.id==="explore"){const b=traitBonus(mouse.trait,"explore");return b>0?`scout + threat −${1+b}`:b<0?"scout (risky)":"discover location";}
  return "";
}

function ResourceBar({s}){
  const season=getSeason(s.turn);
  const items=[{label:"FOOD",val:s.food,cap:s.foodCap,sym:"⁂"},{label:"WOOD",val:s.wood,cap:s.woodCap,sym:"⊞"},{label:"MATS",val:s.mats,cap:s.matsCap,sym:"◈"},{label:"MORALE",val:s.morale,cap:100,sym:"♡"},{label:"THREAT",val:s.threat,cap:10,sym:"!",danger:true}];
  return(<div style={{marginBottom:10}}>
    <div style={{fontFamily:sansInk,fontSize:13,fontWeight:"bold",color:s.turn<=15?C.green:s.turn<=30?C.gold:C.red,marginBottom:5}}>{season.label.toUpperCase()}</div>
    <div style={{display:"grid",gridTemplateColumns:"repeat(5,1fr)",gap:6}}>
      {items.map(({label,val,cap,sym,danger})=>(
        <InkBox key={label} style={{padding:"9px 10px",textAlign:"center"}} fill={danger&&val>=7?C.parchmentDark:C.parchment}>
          <div style={{fontFamily:sansInk,fontSize:11,fontWeight:"bold",color:danger&&val>=7?C.red:C.inkFaded,marginBottom:2}}>{sym} {label}</div>
          <div style={{fontFamily:inkFont,fontSize:20,fontWeight:"bold",color:danger&&val>=7?C.red:C.ink}}>{Math.floor(val)}<span style={{fontSize:13,color:C.inkGhost}}>/{cap}</span></div>
          <div style={{marginTop:4,height:4,background:C.stain}}><div style={{height:"100%",width:`${Math.min(100,val/cap*100)}%`,background:danger?(val>=7?C.red:C.inkFaded):C.inkLight}}/></div>
        </InkBox>
      ))}
    </div>
  </div>);
}

function WinterCheck({s}){
  return(<div style={{display:"grid",gridTemplateColumns:"repeat(4,1fr)",gap:6}}>
    {[{label:"Food",val:s.food,req:30,sym:"⁂"},{label:"Wood",val:s.wood,req:20,sym:"⊞"},{label:"Mats",val:s.mats,req:15,sym:"◈"},{label:"Morale",val:s.morale,req:30,sym:"♡"}].map(r=>(
      <div key={r.label} style={{textAlign:"center",padding:"9px 6px",border:`2px solid ${r.val>=r.req?C.green:C.red}`,background:C.parchment,boxShadow:`2px 2px 0 ${r.val>=r.req?C.green:C.red}`}}>
        <div style={{fontFamily:sansInk,fontSize:12,fontWeight:"bold",color:C.inkFaded}}>{r.sym} {r.label}</div>
        <div style={{fontFamily:inkFont,fontSize:17,fontWeight:"bold",color:r.val>=r.req?C.green:C.red}}>{Math.floor(r.val)}<span style={{fontSize:12,opacity:0.7}}>/{r.req}</span></div>
      </div>
    ))}
  </div>);
}

function FactionBar({factions}){
  const rel=v=>v<=-2?"hostile":v===-1?"wary":v===0?"neutral":v===1?"friendly":"allied";
  const col=v=>v<0?C.red:v===0?C.inkFaded:v===1?C.green:C.gold;
  return(<div style={{display:"grid",gridTemplateColumns:"repeat(4,1fr)",gap:6,marginBottom:10}}>
    {[{id:"spider",label:"Spider",sym:"◉"},{id:"ants",label:"Ants",sym:"⋈"},{id:"patrol",label:"Patrol",sym:"⊞"},{id:"shrine",label:"Shrine",sym:"△"}].map(f=>(
      <div key={f.id} style={{textAlign:"center",padding:"7px 4px",border:`2px solid ${C.ink}`,background:C.parchment,boxShadow:`2px 2px 0 ${C.ink}`}}>
        <div style={{fontFamily:sansInk,fontSize:11,fontWeight:"bold",color:C.inkFaded}}>{f.sym} {f.label.toUpperCase()}</div>
        <div style={{fontFamily:sansInk,fontSize:13,fontWeight:"bold",color:col(factions[f.id])}}>{rel(factions[f.id])}</div>
      </div>
    ))}
  </div>);
}

function IntroScreen({onContinue}){
  const [page,setPage]=useState(0);
  const pages=[
    {body:`The world is very large.\n\nYou have always known this — in the way that a mouse always knows it, which is to say: in the bones, in the whiskers, in the particular quality of stillness that comes before running.\n\nThe world is very large, and most of it does not care about you.`},
    {body:`The Old Giants built it. Nobody remembers them, exactly, but their ruins are everywhere — vast flat plains of grey stone, towers of rusted metal, caverns of glass and wire that hum with something that is not quite alive.\n\nThey left. Or they changed into something else. Or they are still here and simply do not see you.\n\nIt comes to the same thing.`},
    {body:`There are things in this world that hunt.\n\nThe Cat, who moves with the patience of a god who has nothing to prove.\n\nThe Owl, who is older than the Cat, older than the garden, older perhaps than anything that still moves.\n\nThe Fox, who is clever in a way that feels personal.\n\nYou have learned to live between them. Most seasons, this is enough.`},
    {body:`But the summer is over.\n\nYou felt it first in the mornings — a quality of light, a sharpness in the air that wasn't there last week. The leaves are beginning. The nights are lasting longer than they did.\n\nWinter is not a rumour anymore. It is a fact on its way.`},
    {body:`Your village is called Willowroot.\n\nIt is four mice in a burrow under the garden's north wall, a tallow candle stub, a list of supplies that is shorter than you would like, and the particular stubborn warmth of small creatures who have decided to survive.\n\nIt is not much.\n\nIt is enough to begin with.`},
    {body:`You have until the first deep frost.\n\nGather food. Haul timber. Collect the things the Old Giants leave behind — wire and cork and thread and all the small useful objects of a world built for hands much larger than yours.\n\nExplore the garden. Make allies. Make decisions.\n\nSome of them will be wrong. That is permitted. That is, in fact, the nature of decisions.`},
    {heading:"Willowroot",body:`The world is very large.\n\nBut you are here.\nAnd winter can be survived.\nAnd that is, when you think about it carefully, quite a lot.`},
  ];
  const cur=pages[page];const isLast=page===pages.length-1;
  return(<div style={{background:C.parchment,minHeight:420,display:"flex",flexDirection:"column"}}>
    <HeaderSVG/>
    <div style={{flex:1,padding:"1.5rem 1.25rem",display:"flex",flexDirection:"column",justifyContent:"space-between"}}>
      <div style={{display:"flex",gap:6,justifyContent:"center",marginBottom:20}}>
        {pages.map((_,i)=>(<div key={i} style={{width:i===page?20:8,height:8,background:i===page?C.ink:C.stain,transition:"all 0.3s"}}/>))}
      </div>
      <div style={{flex:1,display:"flex",flexDirection:"column",justifyContent:"center"}}>
        {cur.heading&&<div style={{fontFamily:inkFont,fontSize:32,fontWeight:"bold",fontStyle:"italic",color:C.ink,textAlign:"center",marginBottom:20}}>{cur.heading}</div>}
        <div style={{fontFamily:inkFont,fontSize:17,fontStyle:"italic",color:C.inkLight,lineHeight:2.0,textAlign:"center",maxWidth:500,margin:"0 auto",whiteSpace:"pre-line"}}>{cur.body}</div>
      </div>
      <div style={{display:"flex",justifyContent:"space-between",alignItems:"center",marginTop:24}}>
        {page>0?<InkBtn onClick={()=>setPage(p=>p-1)} style={{fontSize:13}}>← back</InkBtn>:<div/>}
        <InkBtn onClick={()=>isLast?onContinue():setPage(p=>p+1)} style={{fontSize:14,padding:"11px 24px",boxShadow:`3px 3px 0 ${C.ink}`}}>{isLast?"❧ Enter Willowroot":"continue →"}</InkBtn>
      </div>
      {!isLast&&<div style={{textAlign:"center",marginTop:12}}><button onClick={onContinue} style={{fontFamily:sansInk,fontSize:12,fontWeight:"bold",background:"none",border:"none",color:C.inkGhost,cursor:"pointer",letterSpacing:"0.06em"}}>skip introduction</button></div>}
    </div>
  </div>);
}

function VillageTab({s,availActions,assign}){
  const [exp,setExp]=useState(null);
  return(<div>
    <Label style={{marginBottom:12}}>Assign each mouse a task for this turn.</Label>
    {s.mice.filter(m=>!m.lost).map(m=>{
      const trait=TRAITS.find(t=>t.id===m.trait)||{label:"?",glyph:"?",desc:""};const isE=exp===m.id;
      return(<InkBox key={m.id} style={{marginBottom:10}} fill={m.injured?C.parchmentDark:C.parchment}>
        <div style={{display:"flex",alignItems:"flex-start",gap:12,marginBottom:10,cursor:"pointer"}} onClick={()=>setExp(isE?null:m.id)}>
          <MouseSVG injured={m.injured} lost={false}/>
          <div style={{flex:1}}>
            <div style={{display:"flex",alignItems:"center",gap:7,flexWrap:"wrap",marginBottom:3}}>
              <Title size={17}>{m.name}</Title>
              <span style={{fontFamily:sansInk,fontSize:12,fontWeight:"bold",padding:"3px 9px",border:`2px solid ${C.ink}`,color:C.inkLight}}>{trait.glyph} {trait.label}</span>
              {m.injured&&<span style={{fontFamily:sansInk,fontSize:12,fontWeight:"bold",padding:"3px 9px",border:`2px solid ${C.red}`,color:C.red}}>✗ injured</span>}
              <span style={{fontFamily:sansInk,fontSize:12,color:C.inkGhost,marginLeft:"auto"}}>{isE?"▲ hide":"▼ details"}</span>
            </div>
            <Label style={{fontSize:12}}>{trait.desc}</Label>
          </div>
        </div>
        {isE&&(<div style={{marginBottom:10,padding:"11px 13px",background:C.parchmentDark,border:`1.5px solid ${C.stain}`}}>
          <Label style={{marginBottom:8,fontSize:12}}>ACTION YIELDS THIS TURN</Label>
          <div style={{display:"grid",gridTemplateColumns:"1fr 1fr",gap:6}}>{availActions.filter(ac=>!(m.injured&&ac.id!=="rest")).map(act=>(<div key={act.id} style={{fontFamily:sansInk,fontSize:13,color:C.inkLight}}><span style={{fontWeight:"bold"}}>{act.glyph} {act.label}:</span>{" "}<span style={{color:C.inkFaded}}>{getActionYield(act,m,s)}</span></div>))}</div>
          {(m.history||[]).length>0&&(<div style={{marginTop:10,paddingTop:10,borderTop:`1.5px solid ${C.stain}`}}><Label style={{marginBottom:5,fontSize:12}}>HISTORY</Label>{m.history.map((h,i)=><Body key={i} style={{fontSize:14}}>· {h}</Body>)}</div>)}
        </div>)}
        <div style={{display:"flex",flexWrap:"wrap",gap:6}}>{availActions.map(act=>(<InkBtn key={act.id} active={s.assignments[m.id]===act.id} disabled={m.injured&&act.id!=="rest"} onClick={()=>assign(m.id,act.id)} style={{fontSize:13,padding:"7px 12px"}}>{act.glyph} {act.label}</InkBtn>))}</div>
      </InkBox>);
    })}
    {s.mice.filter(m=>m.lost).map(m=>(<InkBox key={m.id} style={{marginBottom:10,opacity:0.7}} fill={C.parchmentDark}><div style={{display:"flex",alignItems:"center",gap:12}}><MouseSVG injured={false} lost={true}/><div><div style={{display:"flex",gap:7,alignItems:"center",marginBottom:3}}><Title size={16}>{m.name}</Title><span style={{fontFamily:sansInk,fontSize:13,fontWeight:"bold",color:C.gold,padding:"3px 9px",border:`2px solid ${C.gold}`}}>away</span></div><Label style={{fontSize:13}}>{m.lostReason} — returns in ~{m.lostTurns} turn{m.lostTurns!==1?"s":""}</Label></div></div></InkBox>))}
  </div>);
}

function BuildTab({s,onQueue}){
  return(<div>
    <Label style={{marginBottom:12}}>STRUCTURES & FACILITIES</Label>
    {s.buildings.map(b=>{const can=s.wood>=b.cost.wood&&s.mats>=b.cost.mats,queued=s.buildQueue===b.id;return(<InkBox key={b.id} style={{marginBottom:8,display:"flex",alignItems:"center",justifyContent:"space-between",gap:12,opacity:b.built?0.6:1}}><div style={{flex:1}}><div style={{display:"flex",alignItems:"center",gap:9,marginBottom:3}}><span style={{fontFamily:sansInk,fontSize:22}}>{b.icon}</span><Title size={16}>{b.name}</Title>{b.built&&<span style={{fontFamily:sansInk,fontSize:13,fontWeight:"bold",color:C.green}}>✓ built</span>}</div><Label style={{fontSize:13,marginBottom:3}}>{b.desc} · ⊞{b.cost.wood} ◈{b.cost.mats}</Label>{b.flavor&&<Body style={{fontSize:14}}>{b.flavor}</Body>}</div>{!b.built&&<InkBtn onClick={()=>onQueue(b.id)} disabled={!can} active={queued} style={{fontSize:12,whiteSpace:"nowrap"}}>{queued?"queued":"queue"}</InkBtn>}</InkBox>);})}
  </div>);
}

function MapTab({s,selectedLoc,setSelectedLoc}){
  return(<div>
    <Label style={{marginBottom:8}}>
      Hexes reveal as scouts explore. Click a location for details.
      <span style={{marginLeft:8,color:C.gold}}>{(s.hexMap?.revealed||[]).length}/{HCOLS*HROWS} revealed</span>
    </Label>
    <InkBox style={{padding:"6px",overflow:"hidden"}}><HexMap s={s} allLocs={ALL_LOCATIONS} onHexClick={setSelectedLoc}/></InkBox>
    {selectedLoc&&(<InkBox style={{marginTop:10}} fill={C.parchmentDark}>
      <div style={{display:"flex",justifyContent:"space-between",alignItems:"flex-start"}}>
        <div style={{flex:1}}>
          <div style={{display:"flex",gap:7,alignItems:"center",marginBottom:7,flexWrap:"wrap"}}>
            <Title size={16}>{selectedLoc.name}</Title>
            {selectedLoc.danger&&<span style={{fontFamily:sansInk,fontSize:12,fontWeight:"bold",padding:"3px 9px",border:`2px solid ${C.red}`,color:C.red}}>DANGER</span>}
            {selectedLoc.fluff&&<span style={{fontFamily:sansInk,fontSize:12,fontWeight:"bold",padding:"3px 9px",border:`2px solid ${C.stain}`,color:C.inkFaded}}>FLUFF</span>}
            {selectedLoc.chain&&<span style={{fontFamily:sansInk,fontSize:12,fontWeight:"bold",padding:"3px 9px",border:`2px solid ${C.gold}`,color:C.gold}}>CHAIN</span>}
          </div>
          <Label style={{marginBottom:8,fontSize:13}}>
            STATE: {(s.locationStates?.[selectedLoc.id]||"calm").toUpperCase()}
            {(s.exploredLocs||[]).includes(selectedLoc.id)&&<span style={{marginLeft:8,color:C.green}}>✓ visited</span>}
          </Label>
          <Body style={{fontSize:14}}>{selectedLoc.desc?.[s.locationStates?.[selectedLoc.id]||"calm"]||selectedLoc.desc?.calm||""}</Body>
        </div>
        <button onClick={()=>setSelectedLoc(null)} style={{background:"none",border:"none",color:C.inkFaded,cursor:"pointer",fontSize:22,padding:"0 0 0 12px",lineHeight:1}}>×</button>
      </div>
    </InkBox>)}
  </div>);
}

function LogTab({log}){
  const [open,setOpen]=useState(null);
  return(<div style={{display:"flex",flexDirection:"column",gap:6,maxHeight:420,overflowY:"auto"}}>
    {[...log].reverse().map((e,i)=>{
      const isO=open===i,isF=e.fluff;
      const bg=isF?"#f0ece0":e.good?"#e8f0e0":"#f0e0e0",bc=isF?C.stain:e.good?C.green:C.red;
      return(<div key={i} onClick={()=>setOpen(isO?null:i)} style={{fontFamily:sansInk,fontSize:14,fontWeight:"bold",padding:"9px 13px",background:bg,border:`2px solid ${bc}`,color:isF?C.inkFaded:bc,cursor:"pointer",userSelect:"none",boxShadow:`1px 1px 0 ${bc}`}}>
        <div style={{display:"flex",justifyContent:"space-between",gap:6}}>
          <span><span style={{opacity:0.55,marginRight:8}}>T{e.t}</span>{e.msg}</span>
          <span style={{flexShrink:0}}>{isO?"▲":"▼"}</span>
        </div>
        {isO&&(<div style={{marginTop:9,paddingTop:9,borderTop:`1.5px solid ${bc}`}}>
          {e.title&&<Title size={16} style={{marginBottom:6,color:isF?C.inkFaded:e.good?C.inkLight:C.inkFaded}}>{e.title}</Title>}
          <Body style={{fontSize:15,color:isF?C.inkFaded:e.good?C.inkLight:C.inkFaded}}>{e.lore||"The day passed as days do."}</Body>
        </div>)}
      </div>);
    })}
  </div>);
}

function HelpTab(){
  const [open,setOpen]=useState(null);
  const secs=[
    {id:"goal",  title:"The Goal",        body:"Survive 50 turns and meet winter threshold: 30 food, 20 wood, 15 mats, 30 morale. The world gets harder as winter approaches."},
    {id:"season",title:"Seasonal Pressure",items:[["Turns 1–15","Early Autumn — calm, low threat growth."],["Turns 16–30","Late Autumn — rising pressure, more bad events."],["Turns 31–50","Pre-Winter — brutal threat growth. Prepare early."]]},
    {id:"map",   title:"World Map",       body:"The hex map reveals as scouts explore. Each exploration uncovers a hex and its neighbours. Click revealed location hexes to read details."},
    {id:"factions",title:"Factions",      body:"Spider, Ants, Patrol mice, Shrine keeper. Relations range from hostile to allied. Allied factions reduce threat passively."},
    {id:"chains",title:"Story Chains",    body:"Four multi-step narrative arcs: The Spider's Debt, The Ant Accord, The Shrine Keeper's Trial, The Settlement Mystery."},
    {id:"act",   title:"Actions",         items:ACTIONS.map(a=>[`${a.glyph} ${a.label}`,a.desc])},
    {id:"bldg",  title:"Buildings",       items:ALL_BUILDINGS.map(b=>[`${b.icon} ${b.name}`,`${b.desc} · ⊞${b.cost.wood} ◈${b.cost.mats}`])},
  ];
  return(<div style={{display:"flex",flexDirection:"column",gap:8}}>
    {secs.map((sec,i)=>{const isO=open===i;return(<InkBox key={sec.id} style={{padding:"11px 15px"}}>
      <div onClick={()=>setOpen(isO?null:i)} style={{display:"flex",justifyContent:"space-between",cursor:"pointer",userSelect:"none"}}>
        <Title size={16}>{sec.title}</Title>
        <span style={{fontFamily:sansInk,fontSize:14,fontWeight:"bold",color:C.inkFaded}}>{isO?"▲":"▼"}</span>
      </div>
      {isO&&(<div style={{marginTop:10,borderTop:`1.5px solid ${C.stain}`,paddingTop:10}}>
        {sec.body&&<p style={{fontFamily:sansInk,fontSize:14,fontWeight:"bold",color:C.inkFaded,lineHeight:1.75,margin:0}}>{sec.body}</p>}
        {sec.items&&(<div style={{display:"flex",flexDirection:"column",gap:9}}>{sec.items.map(([lbl,desc])=>(<div key={lbl} style={{display:"flex",gap:12}}><span style={{fontFamily:sansInk,fontSize:13,fontWeight:"bold",color:C.ink,minWidth:140,flexShrink:0}}>{lbl}</span><span style={{fontFamily:sansInk,fontSize:13,color:C.inkFaded,lineHeight:1.65}}>{desc}</span></div>))}</div>)}
      </div>)}
    </InkBox>);})}
  </div>);
}

function Modal({children,wide=false}){
  return(<div style={{minHeight:260,background:"rgba(26,20,16,0.75)",display:"flex",alignItems:"center",justifyContent:"center",padding:16,marginTop:12}}>
    <div style={{background:C.parchment,border:`3px solid ${C.ink}`,boxShadow:`5px 5px 0 ${C.ink}`,padding:24,maxWidth:wide?580:480,width:"100%",position:"relative"}}>
      <div style={{position:"absolute",inset:7,border:`1.5px solid ${C.stain}`,pointerEvents:"none"}}/>
      {children}
    </div>
  </div>);
}

function OutcomePreview({outcomes,state,s}){
  const pool=outcomes?.[state]||outcomes?.calm||[];if(!pool.length)return null;
  const good=pool.filter(o=>o.type==="good"||o.type==="fluff"),bad=pool.filter(o=>o.type==="bad");
  const tot=pool.reduce((a,o)=>a+(typeof o.w==="function"?o.w(s):o.w),0);
  const gPct=Math.round(good.reduce((a,o)=>a+(typeof o.w==="function"?o.w(s):o.w),0)/tot*100);
  return(<div style={{marginBottom:13,padding:"10px 13px",background:C.parchmentDark,border:`2px solid ${C.stain}`}}>
    <Label style={{marginBottom:8,fontSize:12}}>POSSIBLE OUTCOMES — {gPct}% favourable</Label>
    <div style={{display:"flex",flexDirection:"column",gap:5}}>
      {good.map((o,i)=><div key={i} style={{fontFamily:sansInk,fontSize:13,fontWeight:"bold",color:o.type==="fluff"?C.inkFaded:C.green}}>{o.type==="fluff"?"◦":"✓"} {o.title}</div>)}
      {bad.map((o,i)=><div key={i} style={{fontFamily:sansInk,fontSize:13,fontWeight:"bold",color:C.red}}>✗ {o.title}</div>)}
    </div>
  </div>);
}

function ExploreModal({pendingExplore,onEnter,onRetreat,s}){
  const {loc,state}=pendingExplore;
  const desc=loc.desc?.[state]||loc.desc?.calm||loc.desc||"";
  const safe=loc.safe?.[state]||loc.safe?.calm||"Your scouts retreat safely.";
  return(<Modal wide>
    <Label style={{marginBottom:6}}>— SCOUT REPORT — HEX {loc.id} — {state.toUpperCase()} —</Label>
    <div style={{display:"flex",alignItems:"center",gap:9,marginBottom:11,flexWrap:"wrap"}}>
      <Title size={21}>{loc.name}</Title>
      {loc.danger&&<span style={{fontFamily:sansInk,fontSize:12,fontWeight:"bold",padding:"3px 9px",border:`2px solid ${C.red}`,color:C.red}}>DANGEROUS</span>}
      {loc.fluff&&<span style={{fontFamily:sansInk,fontSize:12,fontWeight:"bold",padding:"3px 9px",border:`2px solid ${C.stain}`,color:C.inkFaded}}>FLUFF</span>}
      {state==="corrupted"&&<span style={{fontFamily:sansInk,fontSize:12,fontWeight:"bold",padding:"3px 9px",border:`2px solid ${C.red}`,color:C.red}}>CORRUPTED</span>}
      {state==="disturbed"&&<span style={{fontFamily:sansInk,fontSize:12,fontWeight:"bold",padding:"3px 9px",border:`2px solid ${C.gold}`,color:C.gold}}>DISTURBED</span>}
    </div>
    <Body style={{marginBottom:15,paddingBottom:15,borderBottom:`1.5px solid ${C.stain}`}}>{desc}</Body>
    {loc.outcomes&&<OutcomePreview outcomes={loc.outcomes} state={state} s={s}/>}
    <Label style={{marginBottom:15}}>Safe retreat: {safe}</Label>
    <div style={{display:"flex",gap:10}}>
      <InkBtn onClick={onEnter}   style={{flex:1,padding:"14px 8px",fontSize:14,textAlign:"center"}}>◎ Press Deeper</InkBtn>
      <InkBtn onClick={onRetreat} style={{flex:1,padding:"14px 8px",fontSize:14,textAlign:"center",background:C.parchmentDark}}>☽ Retreat Safely</InkBtn>
    </div>
  </Modal>);
}

function ChainModal({pendingChain,onChoice}){
  const chain=CHAINS[pendingChain.chain],step=chain.steps[pendingChain.step];
  return(<Modal wide>
    <Label style={{marginBottom:6}}>— CHAPTER —</Label>
    <Title size={21} style={{marginBottom:13}}>{step.title}</Title>
    <Body style={{marginBottom:19,paddingBottom:17,borderBottom:`1.5px solid ${C.stain}`}}>{step.body}</Body>
    <div style={{display:"flex",flexDirection:"column",gap:10}}>
      {step.choices.map((c,i)=>(
        <div key={i} onClick={()=>onChoice(c)} style={{border:`2.5px solid ${C.ink}`,padding:"14px 16px",cursor:"pointer",background:C.parchment,fontFamily:sansInk,fontSize:15,fontWeight:"bold",color:C.ink,boxShadow:`2px 2px 0 ${C.ink}`}} onMouseEnter={e=>e.currentTarget.style.background=C.parchmentDark} onMouseLeave={e=>e.currentTarget.style.background=C.parchment}>{c.label}</div>
      ))}
    </div>
  </Modal>);
}

function HeaderSVG(){
  return(<svg width="100%" viewBox="0 0 680 84" style={{display:"block",marginBottom:4}}>
    <rect width="680" height="84" fill={C.parchment}/>
    <rect x="6" y="6" width="668" height="72" fill="none" stroke={C.ink} strokeWidth="2.5"/>
    <rect x="12" y="12" width="656" height="60" fill="none" stroke={C.ink} strokeWidth="1"/>
    {[38,58,78,98,582,602,622,642].map((x,i)=>(<g key={i}><line x1={x} y1="6" x2={x} y2="22" stroke={C.ink} strokeWidth="2"/><line x1={x} y1="62" x2={x} y2="78" stroke={C.ink} strokeWidth="2"/></g>))}
    <text x="340" y="44" textAnchor="middle" fontFamily={inkFont} fontSize="25" fontWeight="bold" fontStyle="italic" fill={C.ink}>Willowroot Village</text>
    <text x="340" y="63" textAnchor="middle" fontFamily={sansInk} fontSize="11" letterSpacing="4" fill={C.inkFaded}>A TALE OF MICE &amp; WINTER</text>
  </svg>);
}

function SavePreview(){
  const [info,setInfo]=useState(null);
  useEffect(()=>{loadGame().then(d=>{if(d)setInfo(d);});}, []);
  if(!info)return null;
  const season=getSeason(info.turn);
  return(<InkBox style={{maxWidth:380,margin:"0 auto",padding:"15px 18px",textAlign:"left"}}>
    <Label style={{marginBottom:7}}>SAVED VILLAGE</Label>
    <Title size={16} style={{marginBottom:5}}>Willowroot — Turn {info.turn} of {info.maxTurns}</Title>
    <div style={{fontFamily:sansInk,fontSize:12,fontWeight:"bold",color:info.turn<=15?C.green:info.turn<=30?C.gold:C.red,marginBottom:10}}>{season.name.toUpperCase()}</div>
    <div style={{display:"grid",gridTemplateColumns:"1fr 1fr",gap:6,marginBottom:10}}>
      {[["⁂",Math.floor(info.food),info.foodCap],["⊞",Math.floor(info.wood),info.woodCap],["◈",Math.floor(info.mats),info.matsCap],["♡",Math.floor(info.morale),100]].map(([sym,val,cap])=>(
        <div key={sym} style={{fontFamily:sansInk,fontSize:13,fontWeight:"bold",color:C.inkFaded}}>{sym} {val}<span style={{opacity:0.5}}>/{cap}</span></div>
      ))}
    </div>
    <Label style={{fontSize:13}}>{(info.mice||[]).filter(m=>!m.lost).map(m=>m.name).join(", ")}</Label>
  </InkBox>);
}

function GameOver({s,onRestart}){
  const pass=s.food>=30&&s.wood>=20&&s.mats>=15&&s.morale>=30;
  const allies=Object.entries(s.factions).filter(([,v])=>v>=2).map(([k])=>k);
  return(<div style={{background:C.parchment,padding:"2rem",textAlign:"center"}}>
    <div style={{fontSize:52,marginBottom:14}}>{pass?"🌾":"❄"}</div>
    <Title size={28} style={{marginBottom:12,textAlign:"center"}}>{pass?"Willowroot Endures":"The Long Freeze"}</Title>
    <p style={{fontFamily:sansInk,fontSize:15,fontWeight:"bold",color:C.inkFaded,lineHeight:1.8,maxWidth:460,margin:"0 auto 20px"}}>{pass?"The stores held. The mice huddled warm, and when the first thaw came, they were still there — singing, arguing, and planning the spring.":"Despite their bravery the cold came too soon. But somewhere beneath the roots, small paws begin to stir again..."}</p>
    {allies.length>0&&<p style={{fontFamily:inkFont,fontSize:15,fontStyle:"italic",color:C.inkLight,marginBottom:20}}>Allied with: {allies.join(", ")}.</p>}
    <WinterCheck s={s}/>
    <div style={{marginTop:20,marginBottom:20}}>
      <Label style={{marginBottom:9}}>MOUSE HISTORIES</Label>
      {s.mice.filter(m=>(m.history||[]).length>0).map(m=>(<div key={m.id} style={{fontFamily:sansInk,fontSize:13,color:C.inkFaded,marginBottom:5}}><strong>{m.name}:</strong> {m.history.join(" · ")}</div>))}
    </div>
    <InkBtn onClick={onRestart} style={{marginTop:4,padding:"13px 36px",fontSize:15}}>Begin Again</InkBtn>
  </div>);
}

// ── APP ───────────────────────────────────────────────────────────────────────
export default function App(){
  const [s,setS]           = useState(()=>initState());
  const [tab,setTab]       = useState("mice");
  const [screen,setScreen] = useState("loading");
  const [hasSave,setHasSave]       = useState(false);
  const [saveStatus,setSaveStatus] = useState("");
  const [selectedLoc,setSelectedLoc] = useState(null);

  useEffect(()=>{
    let cancelled=false;
    loadGame().then(d=>{if(!cancelled){setHasSave(!!d);setScreen("menu");}}).catch(()=>{if(!cancelled)setScreen("menu");});
    const fb=setTimeout(()=>{if(!cancelled)setScreen("menu");},3000);
    return()=>{cancelled=true;clearTimeout(fb);};
  },[]);

  useEffect(()=>{
    if(screen!=="game")return;
    saveGame(s);setSaveStatus("saved");
    const t=setTimeout(()=>setSaveStatus(""),2200);return()=>clearTimeout(t);
  },[s,screen]);

  function startNew(){deleteSave();setS(initState());setHasSave(false);setTab("mice");setSelectedLoc(null);setScreen("intro");}
  function continueGame(){loadGame().then(d=>{if(d){setS(d);setSaveStatus("loaded");setScreen("game");}});}
  function returnToMenu(){setScreen("menu");setHasSave(true);}

  const availActions=ACTIONS.filter(a=>!(a.id==="craft"&&!hasBldg(s,"workshop")));

  function assign(mid,act){if(s.phase!=="assign")return;const m=s.mice.find(x=>x.id===mid);if(m?.injured&&act!=="rest")return;setS(p=>({...p,assignments:{...p.assignments,[mid]:act}}));}
  function setQueue(id){setS(p=>({...p,buildQueue:id}));}
  function endTurn(){if(s.phase==="assign")setS(p=>processTurn(p));}

  function resolveExplore(enter){
    setS(p=>{
      let ns={...p};const {loc,state}=ns.pendingExplore;
      ns.hexMap=revealAround(ns.hexMap||{revealed:[]},loc.id);
      ns.exploredLocs=[...new Set([...(ns.exploredLocs||[]),loc.id])];
      if(enter){
        const outcomes=loc.outcomes?.[state]||loc.outcomes?.calm||[];
        if(outcomes.length){
          const out=pickWeighted(outcomes,ns);
          ns=applyOutcome(ns,out,loc.id);
          const isFluff=out.type==="fluff";
          ns.log=[...ns.log,{t:p.turn-1,msg:`${loc.name}: ${out.title}`,good:!isFluff&&out.type==="good",fluff:isFluff,title:out.title,lore:out.lore}];
        }
      } else {
        const safe=loc.safe?.[state]||loc.safe?.calm||"Your scouts retreat safely.";
        ns.log=[...ns.log,{t:p.turn-1,msg:`Scouts pulled back from ${loc.name}.`,good:true,title:`Retreat: ${loc.name}`,lore:safe}];
      }
      ns.pendingExplore=null;return checkNextPhase(ns);
    });
  }

  function resolveEvent(){
    setS(p=>{
      let ns=p.pendingEvent?Effects.fromData(p.pendingEvent)({...p}):{...p};
      if(p.pendingEvent){if(p.pendingEvent.special==="injure")ns=injureRandom(ns);ns.log=[...ns.log,{t:p.turn-1,msg:p.pendingEvent.short,good:p.pendingEvent.type==="good",title:p.pendingEvent.title,lore:p.pendingEvent.lore}];}
      ns.pendingEvent=null;
      if(ns.turn%10===1&&ns.turn>1){ns.policyChoices=POLICIES.filter(pl=>!ns.policies.includes(pl.id)).sort(()=>Math.random()-0.5).slice(0,3);ns.phase="policy";}
      else if(ns.turn>ns.maxTurns)ns.phase="gameover";
      else ns.phase="assign";
      return ns;
    });
  }

  function resolveChain(choice){setS(p=>checkNextPhase(choice.effect({...p,pendingChain:null,phase:"assign"})));}

  function choosePolicy(pol){
    setS(p=>{let ns=applyPolicyImmediate({...p,policies:[...p.policies,pol.id],phase:"assign",policyChoices:[]},pol);ns.log=[...ns.log,{t:ns.turn,msg:`Policy: "${pol.name}"`,good:true,title:pol.name,lore:pol.flavor}];return ns;});
  }

  if(screen==="loading") return(<div style={{background:C.parchment,minHeight:300,display:"flex",alignItems:"center",justifyContent:"center"}}><Label style={{fontSize:15}}>Checking the burrow...</Label></div>);

  if(screen==="intro") return <IntroScreen onContinue={()=>setScreen("game")}/>;

  if(screen==="menu") return(<div style={{background:C.parchment,minHeight:400}}>
    <HeaderSVG/>
    <div style={{padding:"1.5rem 1rem",textAlign:"center"}}>
      <Body style={{fontSize:17,marginBottom:28,lineHeight:2.0}}>The world is vast and does not care about mice.<br/>But mice care about each other.</Body>
      <div style={{display:"flex",flexDirection:"column",gap:12,maxWidth:360,margin:"0 auto 24px"}}>
        {hasSave&&<InkBtn onClick={continueGame} style={{padding:"16px",fontSize:16,width:"100%",boxShadow:`3px 3px 0 ${C.ink}`}}>❧ Continue Saved Game</InkBtn>}
        <InkBtn onClick={startNew} style={{padding:"16px",fontSize:16,width:"100%",background:hasSave?C.parchmentDark:C.parchment,boxShadow:`3px 3px 0 ${C.ink}`}}>{hasSave?"⊞ New Game (overwrites save)":"⊞ Begin — A New Village"}</InkBtn>
      </div>
      {hasSave&&<SavePreview/>}
      <Label style={{marginTop:24,fontSize:12,color:C.inkGhost}}>Progress saves automatically each turn.</Label>
    </div>
  </div>);

  if(s.phase==="gameover") return(<div style={{background:C.parchment,minHeight:400}}><GameOver s={s} onRestart={()=>{deleteSave();setHasSave(false);setScreen("menu");setS(initState());}}/></div>);

  return(<div style={{background:C.parchment,padding:"0 0 1.5rem",color:C.ink}}>
    <HeaderSVG/>
    <div style={{padding:"0 12px"}}>
      <div style={{display:"flex",justifyContent:"space-between",alignItems:"center",marginBottom:10}}>
        <button onClick={returnToMenu} style={{fontFamily:sansInk,fontSize:13,fontWeight:"bold",background:"none",border:"none",color:C.inkFaded,cursor:"pointer",padding:0}}>← Menu</button>
        <div style={{fontFamily:sansInk,fontSize:12,fontWeight:"bold",color:saveStatus==="saved"?C.green:saveStatus==="loaded"?C.gold:"transparent",transition:"color 0.4s"}}>{saveStatus==="saved"?"✓ saved":saveStatus==="loaded"?"✓ loaded":"-"}</div>
      </div>

      {/* Turn progress bar */}
      <div style={{marginBottom:12,padding:"10px 14px",background:C.parchmentDark,border:`2.5px solid ${C.ink}`,boxShadow:`3px 3px 0 ${C.ink}`}}>
        <div style={{display:"flex",justifyContent:"space-between",marginBottom:7}}>
          <Label>SEASON PROGRESS</Label>
          <Label>TURN {s.turn} / {s.maxTurns}</Label>
        </div>
        <div style={{height:10,background:C.stain,border:`1.5px solid ${C.ink}`,position:"relative"}}>
          <div style={{position:"absolute",top:0,left:0,height:"100%",width:`${(s.turn-1)/s.maxTurns*100}%`,background:s.turn<=15?C.green:s.turn<=30?C.gold:C.red,transition:"width 0.5s"}}/>
          <div style={{position:"absolute",top:"50%",left:"30%",width:"1.5px",height:"100%",background:C.inkFaded,opacity:0.4,transform:"translateY(-50%)"}}/>
          <div style={{position:"absolute",top:"50%",left:"60%",width:"1.5px",height:"100%",background:C.inkFaded,opacity:0.4,transform:"translateY(-50%)"}}/>
          <div style={{position:"absolute",top:"-5px",right:-2,fontFamily:sansInk,fontSize:18}}>❄</div>
        </div>
      </div>

      <ResourceBar s={s}/>
      <FactionBar factions={s.factions}/>

      <div style={{marginBottom:13}}>
        <Label style={{marginBottom:7}}>WINTER READINESS</Label>
        <WinterCheck s={s}/>
      </div>

      {/* Tab nav */}
      <div style={{display:"flex",gap:5,marginBottom:12}}>
        {[["mice","Mice"],["build","Buildings"],["map","Map"],["log","Chronicle"],["help","Help"]].map(([id,lbl])=>(
          <InkBtn key={id} active={tab===id} onClick={()=>setTab(id)} style={{fontSize:13,flex:1,textAlign:"center",padding:"9px 2px"}}>{lbl}</InkBtn>
        ))}
      </div>

      {tab==="mice"  &&<VillageTab s={s} availActions={availActions} assign={assign}/>}
      {tab==="build" &&<BuildTab   s={s} onQueue={setQueue}/>}
      {tab==="map"   &&<MapTab     s={s} selectedLoc={selectedLoc} setSelectedLoc={setSelectedLoc}/>}
      {tab==="log"   &&<LogTab     log={s.log}/>}
      {tab==="help"  &&<HelpTab/>}

      {s.phase==="assign"&&(
        <InkBtn onClick={endTurn} style={{width:"100%",marginTop:12,padding:"15px",fontSize:15,boxShadow:`4px 4px 0 ${C.ink}`}}>
          ❧ End Turn {s.turn} — let the season turn ❧
        </InkBtn>
      )}
    </div>

    {s.phase==="explore"&&s.pendingExplore&&<ExploreModal pendingExplore={s.pendingExplore} onEnter={()=>resolveExplore(true)} onRetreat={()=>resolveExplore(false)} s={s}/>}
    {s.phase==="chain" &&s.pendingChain   &&<ChainModal   pendingChain={s.pendingChain} onChoice={resolveChain}/>}

    {s.phase==="event"&&s.pendingEvent&&(
      <Modal>
        <Label style={{marginBottom:7}}>{s.pendingEvent.type==="good"?"— FORTUNE —":"— ILL OMEN —"}</Label>
        <Title size={21} style={{marginBottom:13}}>{s.pendingEvent.title}</Title>
        <div style={{marginBottom:15,padding:"11px 14px",background:s.pendingEvent.type==="good"?"#e8f0e0":"#f0e0e0",border:`2px solid ${s.pendingEvent.type==="good"?C.green:C.red}`}}>
          <span style={{fontFamily:sansInk,fontSize:16,fontWeight:"bold",color:s.pendingEvent.type==="good"?C.green:C.red}}>{s.pendingEvent.type==="good"?"✓":"✗"} {s.pendingEvent.short}</span>
        </div>
        <Body style={{marginBottom:19}}>{s.pendingEvent.lore}</Body>
        <InkBtn onClick={resolveEvent} style={{width:"100%",padding:"13px",fontSize:15}}>— So it goes —</InkBtn>
      </Modal>
    )}

    {s.phase==="policy"&&(
      <Modal>
        <Label style={{marginBottom:6}}>— VILLAGE COUNCIL —</Label>
        <Title size={21} style={{marginBottom:8}}>Choose a Policy</Title>
        <Body style={{marginBottom:17}}>The elders gather beneath the great root. Three proposals are laid on the moss.</Body>
        <div style={{display:"flex",flexDirection:"column",gap:11}}>
          {s.policyChoices.map(pol=>(
            <div key={pol.id} onClick={()=>choosePolicy(pol)} style={{border:`2.5px solid ${C.ink}`,padding:"14px 16px",cursor:"pointer",background:C.parchment,boxShadow:`2px 2px 0 ${C.ink}`}} onMouseEnter={e=>e.currentTarget.style.background=C.parchmentDark} onMouseLeave={e=>e.currentTarget.style.background=C.parchment}>
              <Title size={17} style={{marginBottom:6}}>{pol.name}</Title>
              <Body style={{marginBottom:9,fontSize:14}}>{pol.flavor}</Body>
              <div style={{display:"flex",gap:16,fontFamily:sansInk,fontSize:13,fontWeight:"bold",flexWrap:"wrap"}}>
                <span style={{color:C.green}}>+ {pol.pos}</span>
                <span style={{color:C.red}}>– {pol.neg}</span>
              </div>
            </div>
          ))}
        </div>
      </Modal>
    )}
  </div>);
}