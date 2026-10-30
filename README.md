@'
import yaml

class GitLabLoader(yaml.SafeLoader):
    pass

GitLabLoader.add_constructor("!reference", lambda loader, node: None)

with open(".gitlab-ci.yml", encoding="utf-8") as f:
    docs = list(yaml.load_all(f, Loader=GitLabLoader))

print("YAML documents:", len(docs))

for job in [".postgres_incremental_guard", "postgres_extract_load_dev_incremental", "postgres_extract_load_ivv_incremental"]:
    locations = [i + 1 for i, d in enumerate(docs) if isinstance(d, dict) and job in d]
    print(f"{job}: {'FOUND in document(s) ' + str(locations) if locations else 'MISSING'}")
'@ | python -
