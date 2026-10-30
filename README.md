@'
import yaml

class GitLabLoader(yaml.SafeLoader):
    pass

def gitlab_reference(loader, node):
    return loader.construct_sequence(node)

GitLabLoader.add_constructor("!reference", gitlab_reference)

with open(".gitlab-ci.yml", encoding="utf-8") as f:
    docs = list(yaml.load_all(f, Loader=GitLabLoader))

print("YAML syntax: OK")
print("Documents:", len(docs))

for job in [
    ".postgres_incremental_guard",
    "postgres_extract_load_dev_incremental",
    "postgres_extract_load_ivw_incremental",
]:
    found = any(isinstance(d, dict) and job in d for d in docs)
    print(f"{job}: {'FOUND' if found else 'MISSING'}")
'@ | python -
