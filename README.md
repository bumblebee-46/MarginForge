# MarginForge — v2.1 calculation reference

## Run and publish

Open the supplied standalone testing index.html directly for a quick test. For the multi-file package, keep index.html and assets/ together and run `python3 -m http.server 8000`, then open http://localhost:8000. Upload this package's contents to your GitHub repository; no build step is required.

The welcome screen runs for 2.5 seconds and fades into the app. The header stays visible while scrolling. Dark is the default theme; theme preference is stored in the browser. The header profile photo links to bumblebee-46's GitHub profile. No repository link appears in the footer.

## Inputs, currencies and engines

All amounts use the selected currency; changing countries does not convert values. Costs are entered excluding recoverable VAT. Amazon supports UK (GBP), France (EUR), and Germany (EUR). eBay, OnCommerce and Shopify use UK (GBP) workbook formulas. TikTok UK follows the separately documented TikTok spreadsheet formula. These are the app's configured calculation rules, not a live marketplace fee feed.

Selling Price Engine finds a price meeting the target margin. Margin Engine evaluates the exact entered selling price. Negative target margins are allowed; targets must be finite and below 100%. Single-product input recalculates after 350 ms. Bulk requires Run Analysis and supports up to 2,000 rows. Templates follow the selected marketplace, country and engine.

Notation: S = VAT-inclusive selling price, C = product cost, F = entered FBA fee, H = shipping, T = storage, Q = quantity, A = advertising percentage / 100. R2(x) means the code's rounding to two decimals. Display formatting does not change the underlying value. Different marketplaces deliberately retain different rounding and margin bases.

## Amazon UK

Only product cost, FBA fee, storage and category enter this calculation. Shipping, delivery, other costs, quantity and advertising are not deducted by this branch. C and F must be positive; storage must be non-negative.

- VAT = S × 20 / 120 (not rounded before deduction).
- Referral = max(round(S × category rate, 2), category minimum).
- Digital fee = round(referral × 2%, 2) + round(F × 2%, 2).
- Profit = round(S − C − VAT − referral − digital fee − F − T, 2).
- Margin % = profit / S × 100.
- Reported total fees = referral + digital fee + F + T (storage is included).

The category rate applies to the entire selling price, rather than progressively to portions of it.

| Category | Rate by selling-price band | Minimum |
| --- | --- | --- |
| Beauty, Health & Personal Care | 8% at S ≤ £10; 15% above | £0.25 |
| Home Products | 8% at S ≤ £20; 15% above | £0.25 |
| Baby Products | 8% at S ≤ £10; 15% above | £0.25 |
| Grocery & Gourmet Foods | 5% at S ≤ £10; 8% above | £0 |
| Clothing & Accessories | 5% at S ≤ £15; 10% above £15 through £20; 15% above £20 | £0.25 |
| Electronic Accessories | 15% | £0.25 |
| Pet Clothing | 5% at S ≤ £10; 15% above | £0.25 |
| Jewellery | 20% at S ≤ £250; 5% above | £0.25 |
| Watches | 15% at S ≤ £50; 8% above £50 through £100; 4% above £100 through £500; 2% above £500 | £0.25 |

Legacy target-price search begins at £2 and follows the original endings, with the original £500 loop bound. Other rounding modes scan from 200 cents through 50,000 cents. Breakeven scans original endings from £0.05 through £500, returning the first price with non-negative rounded profit. The original legacy search can evaluate £500.05 because it snaps the £500 loop value to the next ending.

## Amazon France and Germany

C, F and H must be finite and non-negative; S must be positive. Storage, delivery, quantity, advertising and other costs do not enter these branches. The UK category selector does not apply.

| Rule | France | Germany |
| --- | --- | --- |
| VAT | S × 20 / 120 | S × 19 / 119 |
| Referral | max(€0.30, S × 8%) at S ≤ €10; max(€0.30, S × 15%) above | Same |
| FBA above €12 | F + €0.42 if F > €4; otherwise F + €0.52 | F unchanged |
| Digital rate | 3% | 2% |

At S ≤ €12 France uses F unchanged. Exactly €4 uses the €0.52 increase. There is no weight or volume condition in the implemented model.

- Effective FBA = base FBA + applicable increase.
- Digital referral = R2(referral × digital rate).
- Digital FBA = R2(effective FBA × digital rate).
- Profit = S − C − VAT − referral − effective FBA − digital referral − digital FBA − H.
- Margin % = profit / S × 100.
- Total Amazon fees = referral + effective FBA + both digital components.

VAT, referral and profit retain full precision; only digital components are rounded before deduction. France's optional lowest price in Margin Engine is evaluated independently using the same rules. Export contains the main selling-price result and breakeven, without duplicate item-price or lowest-price columns.

The target-price search treats minimum referral, the €10 referral-rate boundary and France's €12 FBA boundary as separate ranges. It estimates the earliest possible candidate with a rounding allowance, then evaluates actual formulas at each permitted price until the target is met. Its upper bound is €100,000.

## UK workbook channels

S is the total order or pack selling price. Q must be a whole number of at least 1. Advertising percentage must be between 0 and 100. Shipping is deducted once per order.

- VAT = R2(S × 20 / 120).
- Revenue excluding VAT = S − VAT.
- Product and shipping costs = C × Q + H.
- Profit = S − VAT − selling fees − C × Q − H.
- Margin % = profit / (S − VAT) × 100.

Fulfilment, storage, customer delivery and other costs do not enter workbook calculations. Bulk data must leave those fields zero. No additional recoverable fee VAT is deducted except for the explicit eBay advertising multiplier below.

### eBay

- Final value fee = R2(10.9% × S / Q) × Q.
- Regulatory fee = R2(0.35% × S / Q) × Q.
- Below Standard surcharge = R2(6% × S / Q) × Q, only for Below Standard; Top Rated adds zero.
- Fixed order fee = £0.30 at S ≤ £10; £0.40 above £10.
- Advertising fee = R2(R2(S × A) × 1.20).
- Total selling fees = final value + regulatory + Below Standard + fixed order + advertising.

Per-unit rounding occurs before multiplying by quantity. The fixed fee uses the total order price, not the unit price.

### OnCommerce

The internal identifier remains `onbuy` to preserve existing code and template compatibility.

- Sales commission = R2(S × 15%).
- Advertising = R2(S × A).
- Total selling fees = commission + advertising.

### Shopify

- Transaction fee = R2(S × 3% + £0.25).
- Total selling fees = transaction fee.
- Advertising is not included in this branch.

Workbook target-price search estimates an initial candidate from the net-revenue slope, including a rounding allowance. It then evaluates actual rounded calculations at permitted endings until the target is met. eBay searches each fixed-fee band separately. A non-positive slope produces an error; the upper search bound is £100,000.

## Rounding and breakeven

- Next penny / cent evaluates prices on the 0.01 grid.
- Round up to .99 chooses the next price ending .99 at or above the candidate.
- Original endings are .05, .09, .15, .19, .25, .29, .35, .39, .45, .49, .55, .59, .65, .69, .75, .79, .85, .89, .95 and .99.
- Margin Engine does not round the entered price to an ending.
- Breakeven always uses original endings and requires profit ≥ 0. It may show Unavailable if validation fails or no supported price is found within the branch's search range.
- Non-UK-Amazon branches find a zero-target price, then advance if necessary until profit is non-negative.

## Comparison, exports and the breakdown

Marketplace Comparison evaluates each configured channel at the same selling price and current form inputs. It uses the channel-specific rules above; differences in margin denominator are intentional. Channel-specific shipping and other assumptions should be checked in individual calculators.

Bulk exports record the calculation profile, inputs, Selling Price, margin, profit, Breakeven Selling Price and row errors. The Selling Price export value uses the result's customer payment; all currently supported branches have customer payment equal to S. Error rows remain in results. After changing batch inputs or settings, run the analysis again.

The cost bar is a visual summary of existing calculated results. Positive-profit proportions use total displayed components; legend percentages use selling price. For losses, the bar shows shares of total deductions and displays a separate loss notice. Fee-row labels omit parentheses for readability; the removed explanations remain documented here. No design or chart code changes the underlying calculation.

## Implementation map and dormant fallback

`assets/app.js` is the authoritative implementation. Core functions are reproduced below for an exact audit of calculations, candidate searches and rounding. Current supported market profiles select originalAmazonCalc, amazonEUCalc or workbookCalc. The generic calc/findPrice branch is retained for unsupported profiles and is not enabled by the current market selector. Its VAT, minimum commission, payment, extra rate, digital fees and non-recoverable fee-VAT rules are included in the source appendix. The editable profile storage exists but current supported branches use fixed configured formulas.

No live fee data is fetched. Calculations and uploaded product data are processed in the browser. The external font and GitHub avatar may make network requests; theme preference and profile settings may use localStorage.

## TikTok UK

Source: uploaded tiktok cal.xlsx, Sheet1 cells C4 and C7:C9, D7:D15. Selling price is S. Commission = R2(S × 9%); separate additional commission = R2(S × 1%); Smart Promotion = R2(S × entered promotion percentage / 100), default 5%. TikTok shipping is entered separately (sheet example £0.50). Seller shipping is also separate (example £2.40). Product cost is entered per calculation (example £5.89); quantity is not used.

Value after TikTok fees = S − commission − additional commission − Smart Promotion − TikTok shipping. Profit = value after fees − product cost − seller shipping. No VAT is deducted in the supplied formula. Margin, added for the app, = profit / S × 100. At S = £23.99, fee components are £2.16, £0.24 and £1.20, value after fees is £19.89, and profit is £11.60 (48.35% margin).

Target and breakeven use the same evaluated formula and existing rounding choices. Target search estimates the initial candidate with a £0.015 fee-rounding allowance and verifies each permitted price. Search limit is £100,000. Bulk template columns: EAN/ASIN/SKU, Cost, Fulfilment (TikTok shipping), Shipping (seller shipping), SmartPromotionPercent and SellingPrice for Margin Engine. Blank SmartPromotionPercent defaults to 5. No current marketplace fee lookup replaces the supplied rates.

## Exact calculation source

```javascript
const CATS={
  beauty:      {label:"Beauty, Health & Personal Care",hint:"<b>8%</b> if SP ≤ £10 → <b>15%</b> if SP > £10",minFee:0.25,tiers:[{upTo:10,rate:0.08},{upTo:Infinity,rate:0.15}]},
  home:        {label:"Home Products",hint:"<b>8%</b> if SP ≤ £20 → <b>15%</b> if SP > £20",minFee:0.25,tiers:[{upTo:20,rate:0.08},{upTo:Infinity,rate:0.15}]},
  baby:        {label:"Baby Products",hint:"<b>8%</b> if SP ≤ £10 → <b>15%</b> if SP > £10",minFee:0.25,tiers:[{upTo:10,rate:0.08},{upTo:Infinity,rate:0.15}]},
  grocery:     {label:"Grocery & Gourmet Foods",hint:"<b>5%</b> if SP ≤ £10 → <b>8%</b> if SP > £10",minFee:0,tiers:[{upTo:10,rate:0.05},{upTo:Infinity,rate:0.08}]},
  clothing:    {label:"Clothing & Accessories",hint:"<b>5%</b> (≤£15) → <b>10%</b> (£15–£20) → <b>15%</b> (>£20)",minFee:0.25,tiers:[{upTo:15,rate:0.05},{upTo:20,rate:0.10},{upTo:Infinity,rate:0.15}]},
  electronics_acc:{label:"Electronic Accessories",hint:"<b>15%</b> flat on all prices",minFee:0.25,tiers:[{upTo:Infinity,rate:0.15}]},
  pet_clothing:{label:"Pet Clothing",hint:"<b>5%</b> if SP ≤ £10 → <b>15%</b> if SP > £10",minFee:0.25,tiers:[{upTo:10,rate:0.05},{upTo:Infinity,rate:0.15}]},
  jewellery:   {label:"Jewellery",hint:"<b>20%</b> if SP ≤ £250 → <b>5%</b> if SP > £250",minFee:0.25,tiers:[{upTo:250,rate:0.20},{upTo:Infinity,rate:0.05}]},
  watches:     {label:"Watches",hint:"<b>15%</b> (≤£50) → <b>8%</b> (£50–£100) → <b>4%</b> (£100–£500) → <b>2%</b> (>£500)",minFee:0.25,tiers:[{upTo:50,rate:0.15},{upTo:100,rate:0.08},{upTo:500,rate:0.04},{upTo:Infinity,rate:0.02}]},
};
const ENDINGS=[0.05,0.09,0.15,0.19,0.25,0.29,0.35,0.39,0.45,0.49,0.55,0.59,0.65,0.69,0.75,0.79,0.85,0.89,0.95,0.99];
const round = n => Math.sign(n) * Math.round((Math.abs(n) + Number.EPSILON * Math.max(1, Math.abs(n)) * 4) * 100) / 100;

function aFee(sp,k){const t=CATS[k].tiers.find(t=>sp<=t.upTo);return Math.max(Math.round(sp*t.rate*100)/100,CATS[k].minFee)}

function rLbl(sp,k){return(CATS[k].tiers.find(t=>sp<=t.upTo).rate*100).toFixed(0)+'%'}

function dFee(ref,fba){return(Math.round(ref*0.02*100)/100)+(Math.round(fba*0.02*100)/100)}

function mCalc(sp,cost,fba,stor,k){
  const vat=sp*20/120,ref=aFee(sp,k),dst=dFee(ref,fba);
  const pft=Math.round((sp-(cost+vat+ref+dst+fba+stor))*100)/100;
  return{margin:pft/sp,profit:pft,vat,ref,dst};
}

function nextP(p){
  const i=Math.floor(Math.round(p*100)/100),d=Math.round((p-i)*100)/100;
  for(const e of ENDINGS)if(d<=e)return Math.round((i+e)*100)/100;
  return Math.round((i+1+ENDINGS[0])*100)/100;
}

function findSP(cost,fba,tgt,stor,k){
  let sp=2.0;
  while(sp<=500){const c=nextP(sp),r=mCalc(c,cost,fba,stor,k);if(r.margin>=tgt)return{sp:c,...r};sp=Math.round((c+0.01)*100)/100;}
  return null;
}

function defaults(c,m){return{vat:MARKETS[m][1],rate:c==='shopify'?0:15,threshold:c==='amazon'&&m==='UK'?10:0,low:c==='amazon'&&m==='UK'?8:15,min:c==='amazon'?.25:0,fixed:c==='shopify'?.20:0,payment:c==='shopify'?2:0,extra:0,digitalRef:c==='amazon'&&m==='UK'?2:0,digitalFulfil:c==='amazon'&&m==='UK'?2:0,feeVat:20,recover:true,confirmed:false}}

function profile(c=channel){const m=$('market').value;if(c==='amazon'&&['FR','DE'].includes(m))return{channel:c,market:m,amazonEU:true,confirmed:true,vat:m==='FR'?20:19,digital:m==='FR'?3:2,rate:15,threshold:10,low:8,min:.30,payment:0,extra:0,fixed:0,recover:true};if(c==='amazon'&&m==='UK'){const cat=CATS[$('amazonCategory').value];return{channel:c,originalAmazon:true,category:$('amazonCategory').value,vat:20,confirmed:true,rate:cat.tiers.at(-1).rate*100,threshold:0,min:cat.minFee,payment:0,extra:0,fixed:0,recover:true};}if(m==='UK'&&c!=='amazon')return{...defaults(c,m),channel:c,workbook:true,vat:20,rate:c==='ebay'?10.9:c==='onbuy'?15:0,threshold:0,min:0,fixed:c==='shopify'?.25:0,payment:c==='shopify'?3:0,extra:c==='ebay'?.35:0,digitalRef:0,digitalFulfil:0,confirmed:true};return{...defaults(c,m),channel:c,workbook:false,confirmed:false}}

function calc(sp,r,p){if(p.amazonEU)return amazonEUCalc(sp,r,p);if(p.originalAmazon)return originalAmazonCalc(sp,r,p);if(p.workbook)return workbookCalc(sp,r,p);const revenue=round(sp+r.delivery);if(revenue<=0)throw Error('Customer payment must be greater than zero.');const vat=revenue*p.vat/(100+p.vat);const rate=p.threshold>0&&revenue<=p.threshold?p.low:p.rate;const commission=round(Math.max(p.min,revenue*rate/100));const payment=round(revenue*p.payment/100)+p.fixed;const extra=round(revenue*p.extra/100);const digital=round(commission*p.digitalRef/100)+round(r.fulfil*p.digitalFulfil/100);const feeVat=p.recover?0:round((commission+payment+extra+digital+r.fulfil)*p.feeVat/100);const costs=r.cost+r.fulfil+r.shipping+r.storage+r.other;const fees=commission+payment+extra+digital+feeVat;const profit=revenue-vat-costs-fees;return{sp,revenue,vat,rate,commission,payment,extra,digital,feeVat,costs,fees,profit,margin:profit/revenue*100}}

function snap(cents,style){if(style==='99')return Math.floor(cents/100)*100+99>=cents?Math.floor(cents/100)*100+99:(Math.floor(cents/100)+1)*100+99;if(style==='legacy'){const base=Math.floor(cents/100)*100;const e=[5,9,15,19,25,29,35,39,45,49,55,59,65,69,75,79,85,89,95,99].find(e=>base+e>=cents);return e===undefined?base+105:base+e}return cents}

function findPrice(r,p,target,style='cent'){
 if(p.amazonEU)return amazonEUPrice(r,p,target,style);
 if(p.originalAmazon){if(!Number.isFinite(target)||target>=100)throw Error('Enter a valid target margin.');validateAmazonInputs(r);if(style==='legacy'){const res=findSP(r.cost,r.fulfil,target/100,r.storage,p.category);if(!res)throw Error('No valid selling price found under £500.');return originalAmazonCalc(res.sp,r,p);}for(let cents=snap(200,style);cents<=50000;cents=snap(cents+1,style)){const a=originalAmazonCalc(cents/100,r,p);if(a.margin+1e-9>=target)return a;}throw Error('No valid selling price found under £500.');}
 if(p.workbook)return workbookPrice(r,p,target,style);
 if(!Number.isFinite(target)||target>=100)throw Error('Target margin must be below 100%; negative margins are allowed.');
 const limit=10000000;const ranges=p.threshold>0?[[1,Math.min(limit,Math.floor((p.threshold-r.delivery)*100+1e-7))],[Math.max(1,Math.floor((p.threshold-r.delivery)*100+1e-7)+1),limit]]:[[1,limit]];
 const feeTax=p.recover?0:p.feeVat/100;
 const errorBound=.005*((1+p.digitalRef/100)*(1+feeTax)+3*(1+feeTax)+1);
 function possible(c){const revenue=c/100+r.delivery;const rate=p.threshold>0&&revenue<=p.threshold?p.low:p.rate;const commission=Math.max(p.min,revenue*rate/100);const payment=revenue*p.payment/100+p.fixed;const extra=revenue*p.extra/100;const digital=commission*p.digitalRef/100+round(r.fulfil*p.digitalFulfil/100);const fees=(commission+payment+extra+digital)*(1+feeTax)+r.fulfil*feeTax;const costs=r.cost+r.fulfil+r.shipping+r.storage+r.other;return revenue/(1+p.vat/100)-costs-fees+errorBound>=revenue*target/100}
 for(const [start,end] of ranges){if(start>end||!possible(end))continue;let lo=start,hi=end;while(lo<hi){const mid=Math.floor((lo+hi)/2);if(possible(mid))hi=mid;else lo=mid+1}let c=snap(lo,style);while(c<=end){const res=calc(c/100,r,p);if(res.margin+1e-9>=target)return res;c=snap(c+1,style)}}throw Error('No price meets this target within 100,000 currency units. Review costs and fee settings.')}

function breakevenPrice(r,p){try{if(p.originalAmazon){validateAmazonInputs(r);for(let c=snap(1,'legacy');c<=50000;c=snap(c+1,'legacy')){const a=calc(c/100,r,p);if(a.profit>=0)return a;}return null;}let a=findPrice(r,p,0,'legacy');while(a.profit<0){a=calc(snap(Math.round(a.sp*100)+1,'legacy')/100,r,p);}return a;}catch{return null;}}

function validateWorkbookInputs(r,p){if(!Number.isInteger(r.quantity)||r.quantity<1)throw Error('Units must be a whole number of at least 1.');if(!Number.isFinite(r.adRate)||r.adRate<0||r.adRate>100)throw Error('Advertising rate must be between 0 and 100%.');if(p.channel==='ebay'&&!['top','below'].includes(r.sellerStatus))throw Error('Select Top Rated or Below Standard seller status.');}

function workbookCalc(sp,r,p){validateWorkbookInputs(r,p);if(!Number.isFinite(sp)||sp<=0)throw Error('Selling price must be greater than zero.');const vat=round(sp*20/120),netRevenue=sp-vat,quantity=r.quantity,costs=r.cost*quantity+r.shipping;let commission=0,regulatory=0,belowFee=0,fixed=0,advertising=0,payment=0;
 if(p.channel==='ebay'){commission=round(.109*(sp/quantity))*quantity;regulatory=round(.0035*(sp/quantity))*quantity;if(r.sellerStatus==='below')belowFee=round(.06*(sp/quantity))*quantity;fixed=sp<=10?.30:.40;advertising=round(round(sp*r.adRate/100)*1.2);}
 if(p.channel==='onbuy'){commission=round(sp*.15);advertising=round(sp*r.adRate/100);}
 if(p.channel==='shopify')payment=round(sp*.03+.25);
 const fees=commission+regulatory+belowFee+fixed+advertising+payment;const profit=sp-vat-fees-costs;return{workbook:true,channel:p.channel,sp,revenue:sp,netRevenue,vat,quantity,unitCost:r.cost,shipping:r.shipping,adRate:r.adRate,sellerStatus:r.sellerStatus,commission,regulatory,belowFee,fixed,advertising,payment,costs,fees,profit,margin:profit/netRevenue*100};}

function workbookPrice(r,p,target,style){validateWorkbookInputs(r,p);if(!Number.isFinite(target)||target>=100)throw Error('Target margin must be below 100%; negative margins are allowed.');const c=p.channel,ad=c==='shopify'?0:r.adRate/100*(c==='ebay'?1.2:1);const feeRate=c==='ebay'?.109+.0035+(r.sellerStatus==='below'?.06:0):c==='onbuy'?.15:.03;const slope=(1-target/100)/1.2-feeRate-ad;if(slope<=0)throw Error('These fees and advertising costs leave no price that meets this target.');const cost=r.cost*r.quantity+r.shipping;const error=.005*(1+Math.abs(target)/100)+.005*r.quantity*(c==='ebay'?(r.sellerStatus==='below'?3:2):0)+(c==='ebay'?.011:c==='onbuy'?.01:.005);const ranges=c==='ebay'?[[1,1000,.30],[1001,10000000,.40]]:[[1,10000000,c==='shopify'?.25:0]];for(const [lo,hi,fixed] of ranges){let cents=snap(Math.max(lo,Math.ceil(Math.max(0,cost+fixed-error)/slope*100-1e-8)),style);while(cents<=hi){const a=workbookCalc(cents/100,r,p);if(a.margin+1e-9>=target)return a;cents=snap(cents+1,style);}}throw Error('No price meets this target within 100,000 currency units.');}

function validateAmazonInputs(r){if(!Number.isFinite(r.cost)||r.cost<=0||!Number.isFinite(r.fulfil)||r.fulfil<=0||!Number.isFinite(r.storage)||r.storage<0)throw Error('Enter a positive cost and FBA fee, and a non-negative storage cost.');}

function originalAmazonCalc(sp,r,p){validateAmazonInputs(r);if(!Number.isFinite(sp)||sp<=0)throw Error('Selling price must be greater than zero.');const a=mCalc(sp,r.cost,r.fulfil,r.storage,p.category);return{originalAmazon:true,sp,revenue:sp,vat:a.vat,commission:a.ref,digital:a.dst,profit:a.profit,margin:a.margin*100,costs:r.cost+r.fulfil+r.storage,fees:a.ref+a.dst+r.fulfil+r.storage,cost:r.cost,fulfil:r.fulfil,storage:r.storage,category:p.category};}

function validateAmazonEU(r){for(const k of ['cost','fulfil','shipping'])if(!Number.isFinite(r[k])||r[k]<0)throw Error('Enter a valid non-negative '+k+' amount.');}

function amazonEUCalc(sp,r,p){validateAmazonEU(r);if(!Number.isFinite(sp)||sp<=0)throw Error('Selling price must be greater than zero.');const vat=sp*p.vat/(100+p.vat),rate=sp<=10?.08:.15,commission=Math.max(.30,sp*rate);const fbaIncrease=p.market==='FR'&&sp>12?(r.fulfil>4?.42:.52):0;const fba=r.fulfil+fbaIncrease;const digitalReferral=round(commission*p.digital/100),digitalFBA=round(fba*p.digital/100);const digital=digitalReferral+digitalFBA;const profit=sp-r.cost-vat-commission-fba-digital-r.shipping;return{amazonEU:true,market:p.market,sp,revenue:sp,vat,commission,rate:rate*100,digital,digitalReferral,digitalFBA,digitalRate:p.digital,baseFBA:r.fulfil,fbaIncrease,fulfil:fba,cost:r.cost,shipping:r.shipping,fees:commission+fba+digital,costs:r.cost+r.shipping,profit,margin:profit/sp*100};}

function amazonEUPrice(r,p,target,style='cent'){validateAmazonEU(r);if(!Number.isFinite(target)||target>=100)throw Error('Enter a target margin below 100%; negative margins are allowed.');const limit=10000000;const ranges=p.market==='FR'?[[1,375,0],[376,1000,.08],[1001,1200,.15],[1201,limit,.15]]:[[1,375,0],[376,1000,.08],[1001,limit,.15]];for(const [lo,hi,refRate]of ranges){const increase=p.market==='FR'&&lo>1200?(r.fulfil>4?.42:.52):0;const fba=r.fulfil+increase;const minimum=refRate===0?.30:0;const fixed=r.cost+r.shipping+fba+round(fba*p.digital/100)+minimum+(minimum?round(minimum*p.digital/100):0);const slope=1/(1+p.vat/100)-refRate*(1+p.digital/100)-target/100;if(slope<=0)continue;const err=minimum?0:.005;let cents=snap(Math.max(lo,Math.ceil(Math.max(0,fixed-err)/slope*100-1e-8)),style);while(cents<=hi){const a=amazonEUCalc(cents/100,r,p);if(a.margin+1e-9>=target)return a;cents=snap(cents+1,style);}}throw Error('No selling price meets this target within 100,000 EUR.');}
```

### TikTok calculation source
```javascript
function tiktokCalc(sp,r){if(!Number.isFinite(sp)||sp<=0)throw Error('Selling price must be greater than zero.');for(const k of ['cost','fulfil','shipping'])if(!Number.isFinite(r[k])||r[k]<0)throw Error('Enter a valid non-negative '+k+' amount.');if(!Number.isFinite(r.adRate)||r.adRate<0||r.adRate>100)throw Error('Smart Promotion must be between 0 and 100%.');const commission=round(sp*.09),additionalCommission=round(sp*.01),advertising=round(sp*r.adRate/100),fees=commission+additionalCommission+advertising+r.fulfil,costs=r.cost+r.shipping,profit=sp-fees-costs;return{workbook:true,channel:'tiktok',sp,revenue:sp,netRevenue:sp,vat:0,commission,additionalCommission,advertising,fulfil:r.fulfil,shipping:r.shipping,cost:r.cost,costs,fees,profit,margin:profit/sp*100};}
function tiktokPrice(r,target,style){if(!Number.isFinite(target)||target>=100)throw Error('Target margin must be below 100%; negative margins are allowed.');tiktokCalc(1,r);const slope=1-.09-.01-r.adRate/100-target/100;if(slope<=0)throw Error('These fees leave no price that meets this target.');let cents=snap(Math.max(1,Math.ceil(Math.max(0,r.cost+r.shipping+r.fulfil-.015)/slope*100-1e-8)),style);for(;cents<=10000000;cents=snap(cents+1,style)){const a=tiktokCalc(cents/100,r);if(a.margin+1e-9>=target)return a;}throw Error('No price meets this target within £100,000.');}

```

## Saved inputs and fee-model metadata

Single-product fields are stored in browser localStorage under marginforge-product-inputs-v1, separately for each marketplace and country. Switching channels or countries restores that profile's values; new TikTok inputs default to £0.50 TikTok shipping and 5% Smart Promotion. Other new channels begin with zero product/shipping costs. Reset product replaces the current saved inputs and restores the TikTok-specific defaults where applicable. Bulk pasted data and file uploads are not saved by this feature. Browser storage failure falls back to in-session state. Avoid entering confidential data on shared computers.

The app explicitly identifies margin on selling price for Amazon and TikTok, and margin on revenue excluding VAT for eBay, OnCommerce and Shopify. Fee model v2.1 is labelled Configured 4 Oct 2026. This is the application configuration date, not confirmation that seller fees were verified live on that date. These additions do not change any pricing formula.

## Interface languages

English is default. The header offers English, Español, Français and Deutsch. The browser remembers language under marginforge-language. Navigation, common controls, result/fee labels and a translated help guide use local dictionaries; language never changes market, currency or formulas. CSV field names remain in English for compatibility. Long diagnostic and legacy formula notes may retain English where no dictionary entry exists. The creator credit remains bumblebee-46 in About. The profile photograph has been replaced by the language selector.

## Incomplete single-product inputs

New profiles and Reset product leave cost, shipping, target margin and current selling price empty. Amazon profiles also leave FBA empty. Required fields are checked per marketplace and engine: cost and target/current price, plus FBA for Amazon UK; FBA and shipping for Amazon FR/DE; TikTok shipping fee and seller shipping for TikTok; seller shipping for other UK channels. Blank required fields show Awaiting inputs and are highlighted. Explicit zero shipping is valid. Existing saved complete inputs are restored. Invalid numeric values continue to use the original validation rules. Bulk calculations remain manual and unchanged.

## Presentation polish

Result cards prioritize the calculated outcome. In Margin Engine the redundant margin metric is hidden, leaving net profit and total fees below the main margin. Fee values are aligned, Net Profit has a final highlighted row, incomplete fields have a calculator illustration and missing-input checklist. Spacing uses consistent 40 px fields, 16 px layout gaps, and 8 px card corners. On mobile, marketplace tabs scroll horizontally, inputs stack, bulk actions wrap, and tables scroll inside their container. Subtle transitions respect reduced-motion settings. Expanded local translations cover common errors, import prompts, formula notes and About/Privacy. CSV identifiers are preserved. Browser rendering still requires a manual visual check at desktop, tablet and mobile widths; numerical and source/package validation does not establish visual rendering.
