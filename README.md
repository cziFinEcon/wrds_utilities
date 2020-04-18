# Sample SAS Programs for Processing WRDS Data

In this repository, I present a selection of SAS programs that (pre-)process WRDS data.
My goal is to provide efficient procedures for turning raw data from various WRDS databases (e.g., CRSP, Compustat, IBES, etc.) into clean and well-structured datasets, conducive to whatever econometric analysis that may follow.
By walking through the steps in each program, one should be able to 
(1) quickly gain a working knowledge of related data (e.g., file structures, variable definitions, etc.)
and (2) understand the role of each step in the relevant process.
I believe these programs are well-written and should be pretty straightforward to interpret, even for people who are new to SAS.
They are also very flexible and can be easily customized to fit any specific research needs.
Should you have any questions and see any bugs, please submit an issue or email me at czi.academic@gmail.com.
I am happy to help!

#### Table of Contents

- NBER recession indicator (SAS & Stata)
- [Construct a panel data set without time gaps from the Compustat universe](#build_panel)



<a name="build_panel"></a>
### Construct a panel data set without time gaps from the Compustat universe

The goal of this exercise is to construct a panel data set from the [Compustat](https://wrds-web.wharton.upenn.edu/wrds/query_forms/navigation.cfm?navId=60) database that facilitates subsequent analysis.
The resulting panel features a continuous yearly series without gap for each firm, which helps ensure the proper function of time-series operators. 
The detailed procedure is as follows.

Start with `comp.funda`, choose sample period and apply standard filters.
Only keep relevant variables.
The combination of `gvkey` and `datadate` is the key that uniquely identifies each observation.
```sas
%let yr_beg = 1978;
%let yr_end = 2017;
%let funda_filter = ((consol eq 'C') and (indfmt eq 'INDL') and 
                     (datafmt eq 'STD') and (popsrc eq 'D') and
                     (fic eq 'USA') and (at gt 0) and (sale gt 0));
%let secm_filter = (primiss eq 'P' and fic eq 'USA' and curcdm eq 'USD');
%let comp_sample_period = ("01jan&yr_beg."d le datadate le "31dec&yr_end."d);
%let funda_keys = gvkey datadate;
%let funda_vars = conm sich at sale seq ceq pstk lt mib
                  txditc txdb itcb pstkrv pstkl prcc_f 
                  csho dlc dltt capx ppent ppegt invt ib
                  dp cogs xsga lct lo
;

data _tmp1;
set comp.funda;
where &comp_sample_period. 
  and &funda_filter.
;
keep &funda_keys. &funda_vars.;
run;
```

Build a full yearly series without gap for each `gvkey` in the initial sample.
```sas
proc sql;
create table _tmp11 as
select gvkey , 
       min(year(datadate)) as yr_beg ,
       max(year(datadate)) as yr_end
from _tmp1
group by gvkey
;
quit;

%let nyr = %eval(&yr_end. - &yr_beg.);
data _tmp12;
set _tmp11;
yr_0 = &yr_beg.;
array yr {&nyr.} yr_1 - yr_&nyr.;
do i = 1 to &nyr.;
  yr(i) = yr_0 + i;
end;
drop i;
proc transpose 
  data = _tmp12 
  out = _tmp13 (drop = _name_ 
                rename = (col1 = yr)
                where = (yr_beg le yr le yr_end))
;
by gvkey yr_beg yr_end;
var yr_0 - yr_&nyr.;
run;
```

Compute the market value of equity using `prccm` and `cshoq` of the primary issue (`primiss = 'P'`).
Aggregate it to a yearly frequency by keeping only the last non-missing value for each firm-year.
```sas
data _tmp14;
keep gvkey yr datadate me_secm;
set comp.secm;
where &secm_filter.
  and &comp_sample_period.
;
yr = year(datadate);
me_secm = prccm * cshoq;
if not missing(me_secm);
proc sort nodupkey; by gvkey yr datadate;
data _tmp14; set _tmp14;
by gvkey yr datadate;
if last.yr;
run;
```

Combine the information from different Compustat data sets and compute a group of variables.
Use alternative definitions (and variables) in the calculation to reduce missing instances.
```sas
proc sql;
create table _tmp15 as
select a.gvkey , a.yr , b.* , 
       c.me_secm , input(d.sic , 8.) as sicn
from _tmp13 as a
     left join
     _tmp1 as b
     on a.gvkey eq b.gvkey and
        a.yr eq year(b.datadate)
     left join
     _tmp14 as c
     on a.gvkey eq c.gvkey and
        a.yr eq c.yr
     left join
     comp.names as d
     on a.gvkey eq d.gvkey and
        d.year1 le a.yr le d.year2
order by a.gvkey , a.yr , b.datadate
;
quit;

data _tmp2;
format gvkey yr datadate
       conm sic at sale
       be me 
;
keep gvkey yr datadate 
     conm sic at sale
     be me 
;
set _tmp15;
sic = coalesce(sich , sicn);
be = coalesce(seq , ceq + pstk , at - lt - mib)
     + coalesce(txditc , txdb + itcb , 
                lt - lct - lo - dltt , 0)
     - coalesce(pstkrv , pstkl , pstk , 0)
;
me = coalesce(prcc_f * csho , me_secm);
run;
```

Keep only the last observation for each firm-year.
```sas
proc sort data = _tmp2; by gvkey yr datadate;
data _tmp2;
set _tmp2;
by gvkey yr datadate;
if last.yr;
/* check uniqueness of the key */
proc sort nodupkey; by gvkey yr; 
run;
```
