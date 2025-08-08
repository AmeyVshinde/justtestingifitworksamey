import streamlit as st
import pandas as pd
import io
import json
from datetime import datetime

st.set_page_config(page_title="Google Ads Bulk Generator", layout="wide")

st.title("Google Ads — Controlled Bulk Upload Generator")
st.markdown("Executives upload Keywords & Ads. Team Lead supplies settings. App validates and builds a Google Ads Editor CSV.")

# ------------------------- Helpers -------------------------

def load_settings_from_file(uploaded_file):
    if uploaded_file is None:
        return None
    name = uploaded_file.name.lower()
    try:
        if name.endswith('.json'):
            return json.load(uploaded_file)
        else:
            # accept csv with two columns: key,value or a small mapping table
            df = pd.read_csv(uploaded_file)
            return df.to_dict(orient='list')
    except Exception as e:
        st.error(f"Can't parse settings file: {e}")
        return None


def validate_campaign_name(name, pattern_prefix=None):
    # Simple validation: not empty and contains prefix if provided
    if pd.isna(name) or str(name).strip()=="":
        return False, "Campaign name empty"
    if pattern_prefix and not str(name).startswith(pattern_prefix):
        return False, f"Campaign name should start with '{pattern_prefix}'"
    return True, ""


def apply_team_lead_defaults(campaign_name, settings):
    # Resolve mapping from campaign name -> location/audience/budget
    out = {}
    mappings = settings.get('campaign_mappings', {})
    # try exact match then prefix match
    if campaign_name in mappings:
        out.update(mappings[campaign_name])
    else:
        # prefix match
        for prefix, vals in mappings.items():
            if campaign_name.startswith(prefix):
                out.update(vals)
                break
    # fallbacks
    out.setdefault('Location Targeting', settings.get('default_location', 'India'))
    out.setdefault('Language Targeting', settings.get('default_language', 'English'))
    out.setdefault('Budget', settings.get('default_budget', '1000'))
    out.setdefault('Bid Strategy Type', settings.get('default_bid_strategy', 'Maximize conversions'))
    return out


def build_bulk_dataframe(keywords_df, ads_df, settings, pattern_prefix=None):
    rows = []
    errors = []

    # Normalize column names
    kcols = [c.lower() for c in keywords_df.columns]
    acols = [c.lower() for c in ads_df.columns]

    for _, k in keywords_df.iterrows():
        camp = k.get('Campaign') if 'Campaign' in keywords_df.columns else k.get('campaign')
        ag = k.get('Ad group') if 'Ad group' in keywords_df.columns else k.get('ad group')
        if pd.isna(camp) or pd.isna(ag):
            errors.append((k.to_dict(), 'Missing Campaign or Ad group'))
            continue
        ok, msg = validate_campaign_name(camp, pattern_prefix)
        if not ok:
            errors.append((k.to_dict(), msg))
            continue
        defaults = apply_team_lead_defaults(camp, settings)
        row = {
            'Record Type': 'Campaign',
            'Campaign': camp,
            'Campaign Status': 'Enabled',
            'Campaign Type': defaults.get('Campaign Type', 'Search'),
            'Budget': defaults.get('Budget'),
            'Location Targeting': defaults.get('Location Targeting'),
            'Language Targeting': defaults.get('Language Targeting'),
            'Bid Strategy Type': defaults.get('Bid Strategy Type')
        }
        rows.append(row)

        # ad group row
        ag_row = {
            'Record Type': 'Ad group',
            'Campaign': camp,
            'Ad Group': ag,
            'Ad Group Status': 'Enabled',
            'Max CPC': k.get('Max CPC', settings.get('default_max_cpc')) if 'Max CPC' in keywords_df.columns else settings.get('default_max_cpc', ''),
        }
        rows.append(ag_row)

        # keyword row
        kw_row = {
            'Record Type': 'Keyword',
            'Campaign': camp,
            'Ad Group': ag,
            'Keyword': k.get('Keyword') if 'Keyword' in keywords_df.columns else k.get('keyword'),
            'Match Type': k.get('Match Type', 'Phrase') if 'Match Type' in keywords_df.columns else k.get('match type','Phrase'),
            'Max CPC': k.get('Max CPC', settings.get('default_max_cpc',''))
        }
        rows.append(kw_row)

        # attach ads for this ad group (if present)
        matched_ads = ads_df[(ads_df['Campaign'] == camp) & (ads_df['Ad Group'] == ag)] if 'Campaign' in ads_df.columns and 'Ad Group' in ads_df.columns else pd.DataFrame()
        if matched_ads.empty:
            errors.append(({'Campaign': camp, 'Ad group': ag}, 'No ads found for this ad group'))
        else:
            for _, ad in matched_ads.iterrows():
                ad_row = {
                    'Record Type': 'Ad',
                    'Campaign': camp,
                    'Ad Group': ag,
                    'Ad Type': ad.get('Ad Type', 'Responsive search ad') if 'Ad Type' in ad.index else 'Responsive search ad',
                    'Headlines': ad.get('Headlines') if 'Headlines' in ad.index else ad.get('headlines',''),
                    'Descriptions': ad.get('Descriptions') if 'Descriptions' in ad.index else ad.get('descriptions',''),
                    'Final URL': ad.get('Final URL') if 'Final URL' in ad.index else ad.get('final url',''),
                    'Status': ad.get('Status','Enabled')
                }
                rows.append(ad_row)

    bulk_df = pd.DataFrame(rows)
    # de-duplicate campaign rows (keep first)
    if not bulk_df.empty:
        campaigns_mask = bulk_df['Record Type'] == 'Campaign'
        seen = set()
        keep_indexes = []
        for i, row in bulk_df[campaigns_mask].iterrows():
            key = row['Campaign']
            if key not in seen:
                seen.add(key)
                keep_indexes.append(i)
        # remove other campaign rows
        drop_idxs = [i for i, r in bulk_df[campaigns_mask].iterrows() if i not in keep_indexes]
        bulk_df = bulk_df.drop(index=drop_idxs).reset_index(drop=True)

    return bulk_df, errors

# ------------------------- UI -------------------------

st.sidebar.header('Team Lead — Settings')
settings_file = st.sidebar.file_uploader('Upload settings JSON/CSV (optional)', type=['json','csv'])
pattern_prefix = st.sidebar.text_input('Campaign name prefix (optional)', value='LG-2025-')

# Provide a small default settings example
default_settings = {
    "campaign_mappings": {
        "LG-2025-GDC-India": {
            "Location Targeting": "India",
            "Budget": 5000,
            "Campaign Type": "Search",
            "Bid Strategy Type": "Maximize conversions"
        }
    },
    "default_location": "India",
    "default_language": "English",
    "default_budget": 1000,
    "default_bid_strategy": "Maximize conversions",
    "default_max_cpc": 50
}

if settings_file is None:
    st.sidebar.markdown('No settings file uploaded — using built-in defaults')
    settings = default_settings
else:
    st.sidebar.markdown(f'Loaded {settings_file.name}')
    settings = load_settings_from_file(settings_file) or default_settings

st.sidebar.markdown('---')
st.sidebar.markdown('Sample settings JSON schema:')
st.sidebar.code(json.dumps(default_settings, indent=2))

# Executive uploads
st.header('Executive Uploads')
col1, col2 = st.columns(2)
with col1:
    kw_file = st.file_uploader('Upload Keywords file (CSV/Excel). Required columns: Campaign, Ad Group, Keyword, Match Type, Max CPC (optional)', type=['csv','xlsx','xls'])
with col2:
    ads_file = st.file_uploader('Upload Ads file (CSV/Excel). Required columns: Campaign, Ad Group, Headlines, Descriptions, Final URL (optional)', type=['csv','xlsx','xls'])

if st.button('Process & Generate'):
    if kw_file is None or ads_file is None:
        st.error('Please upload both Keywords and Ads files to proceed.')
    else:
        try:
            if kw_file.name.endswith('.csv'):
                kw_df = pd.read_csv(kw_file)
            else:
                kw_df = pd.read_excel(kw_file)
            if ads_file.name.endswith('.csv'):
                ads_df = pd.read_csv(ads_file)
            else:
                ads_df = pd.read_excel(ads_file)

            bulk_df, errors = build_bulk_dataframe(kw_df, ads_df, settings if isinstance(settings, dict) else default_settings, pattern_prefix)

            st.subheader('Preview — Generated Bulk Rows')
            st.dataframe(bulk_df.head(200))

            if errors:
                st.subheader('Validation / Warnings')
                for e in errors[:50]:
                    st.warning(f"{e[1]} — {e[0]}")

            # Provide download
            if not bulk_df.empty:
                csv_bytes = bulk_df.to_csv(index=False).encode('utf-8')
                now = datetime.now().strftime('%Y%m%d_%H%M%S')
                filename = f'google_ads_bulk_{now}.csv'
                st.download_button('Download bulk CSV', data=csv_bytes, file_name=filename, mime='text/csv')

        except Exception as e:
            st.error(f'Error processing files: {e}')

st.markdown('---')
st.caption('This is a starter app. You can extend validation rules, mappings, and UI as needed.')
