# -----------------------------------------------------------------------------
# Optional custom image.
#
# The Kubernetes manifests use the stock `vllm/vllm-openai` image directly,
# which is the right call for most deployments. Build this image only when
# you need extra Python dependencies (custom tokenizers, guided decoding
# backends, private wheels) or want to bake in configuration.
#
#   docker build -t <registry>/inference-server:0.1.0 docker/
#   docker push <registry>/inference-server:0.1.0
#
# Then point spec.template.spec.containers[0].image at your tag (an
# `images:` override in the overlay kustomization is the cleanest way).
# -----------------------------------------------------------------------------
FROM vllm/vllm-openai:v0.9.1

# Extra dependencies go here. Keep the layer count low.
# RUN pip install --no-cache-dir outlines==0.0.46 my-private-package==1.2.3

# Run as non-root; aligns with the pod securityContext (fsGroup 1000).
RUN useradd --create-home --uid 1000 vllm
USER 1000

# Same entrypoint contract as the manifests: args are supplied by the
# Deployment, so this image stays configuration-free.
ENTRYPOINT ["python3", "-m", "vllm.entrypoints.openai.api_server"]
