## Requirements
* vagrant
* virtualbox
* ansible
* tkn (tekton cli client)
* Ports 2222, 6443 and 8080 in host machine
* Port 8080 has to be open and port forwarding configured to receive GitHub webhooks
* Fork of https://github.com/algolia/instant-search-demo and add a Dockerfile. Example in https://github.com/stefanbs/instant-search-demo
* Configure a GitHub webhook pointing to http://<your_public_ip_address>:8080/webhook/ and with Content type application/json
* Create group_vars/vars.yaml file containing the following vars:
  * container_registry: the container registry for the instant-search-demo container image
  * github_repo: github repository for instant-search-demo, forked from the original sent with the assignment. This repo should have a Dockerfile.
  * vault_dockerconfigjson: ansible-vault encrypted version of the base64 encoded contents of your container registry auth file. It's usually found in ~/.docker/config.json after running a docker login command.

Example group_vars/vars.yaml file:
```yaml
---
container_registry: quay.io/sbullone/instant-search-demo:latest
github_repo: https://github.com/StefanBS/instant-search-demo
vault_dockerconfigjson: !vault |
      $ANSIBLE_VAULT;1.1;AES256
      63343563353130396666663463393434303562386333633733336435316631343235393633393133
      6636336661313965653633343138313034653531396162380a656232343132336436363432336533
      61373033363766663930653537306230346630333335333933336262323865303963393737363930
      61373561383535396339343038366238376138666535336230396163356166333264366565366663
      63373435393862633436393731343032663766386537313564633862353266346332363764303635
      66353961313832396466333731643734613865666361663761623138363633663033616362366331
      31643636653564613239316638303337613739353366323832373338336630343331373864373633
      36373966633632326363656535663664666332326236333530616530633961363665316661343731
      37616535663634373766333864326161386462353766636331343231313062343132326639373462
      313230636334373138316339313638373238
```

A k3s (single instance k8s cluster) instance is deployed with Vagrant. Ansible is used to deploy the required software to the cluster. We install tekton-pipelines, tekton-triggers and tekton-interceptors to handle CI/CD, as well as it's CLI client tkn. Ports 2222, 6443 and 8080 need to be available in the host machine for SSH access, Kubernetes API and ingress respectively, and port 8080 has to be open and port forwarding to the host to receive webhooks from GitHub.

instant-search-demo app is deployed and accessible in http://localhost:8080/demo/. Committing changes to your GitHub repo will trigger a new pipeline run in the namespace instant-search-demo. This pipeline will clone the repository, build a container image using the provided Dockerfile, upload it to the container registry and trigger a rolling update of the instant-search-demo app. If the new container version works the rolling update should delete the old pods and keep the new ones. If the new container version doesn't work the rolling update process will keep the old containers running and no downtime will be experienced.

## Deploying
```sh
$ vagrant up --provider virtualbox
$ ansible-playbook -i inventory.yaml --ask-vault-password ansible-deploy.yaml -e @group_vars/vars.yaml
```

New pipeline-runs will be triggered when code is committed to the GitHub repo. To check the pipeline-run with tkn run the following command:

```sh
[sbullones@stefan-laptop test2]$ tkn pipelinerun list -n instant-search-demo
NAME                          STARTED        DURATION   STATUS
tutorial-pipeline-run-wpqq6   1 minute ago   ---        Running
```

To follow the pipeline-run logs:
```sh
[sbullones@stefan-laptop test2]$ tkn pipelinerun logs tutorial-pipeline-run-wpqq6 -f -n instant-search-demo
```
